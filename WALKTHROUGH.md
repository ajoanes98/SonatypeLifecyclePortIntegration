# Building the Sonatype Lifecycle Ocean Integration — Walkthrough

This document explains how the Sonatype Lifecycle integration was built on the
[Port Ocean](https://ocean.port.io/developing-an-integration/) framework: what
we created at each step, the decisions behind the data model, and how a customer
sets it up. It doubles as the design rationale for the pull request that makes
this an official Port integration.

---

## 1. Goal and data model

**Goal:** bring Sonatype Lifecycle (Nexus IQ Server) software-composition data
into Port so that platform and security teams can see, for every application in
the catalog, its open-source risk posture — which policies are violated, at what
severity, on which components, at which lifecycle stage — and act on it from the
same place they manage everything else.

Sonatype's own hierarchy maps cleanly onto Port entities, so we mirror it:

```
Organization
   └── Application
          ├── Report            (one per lifecycle stage: build / stage-release / release / operate)
          │      └── Policy Violation   (one per component × violated policy)
          └── Component          (one per open-source component in a stage report)
                 └── Vulnerability (CVE)   (shared/deduped across every affected component)
```

Relations point up the chain: a policy violation belongs to a report, which
belongs to an application, which belongs to an organization. Components relate to
their application and report, and to the vulnerabilities (CVEs) affecting them;
each component also carries the **recommended upgrade version** Sonatype suggests
as the fix. That lets you start from any level — "show me every critical
violation in the Payments org", or "which services use a component with a fix
available for CVE-2021-44228" — and traverse. Applications can additionally be
linked to your existing Port `service` blueprint (see the example config), so
Sonatype risk shows up directly on each service page.

### Key mapping decisions

- **Identifiers use Sonatype's internal IDs, not public IDs.** IQ's internal
  IDs are stable and are what every other endpoint (reports, relations) uses;
  the friendly `publicId` is kept as a searchable property on the application.
- **One report entity per application *per stage*, updated in place.** We key a
  report on `<applicationInternalId>-<stage>` rather than on the per-scan
  `reportId`. The catalog therefore holds *"the current release report"*,
  *"the current build report"*, etc., and each new scan updates that same entity
  instead of accumulating one entity per scan forever.
- **Policy violations are derived from the policy report, keyed by
  `(reportId, componentHash, policyId)`.** This deterministic key means a
  re-scan updates existing violations in place, and violations that get fixed or
  waived simply stop being reported and are pruned by Ocean at the end of a
  resync.
- **Severity is a named bucket, not a raw number.** Sonatype expresses risk as
  a 0–10 "threat level"; the IQ UI groups those into Critical (8–10), Severe
  (4–7), Moderate (2–3), Low (1). We compute the same buckets so the catalog
  speaks the language operators already know, while also keeping the raw
  `threatLevel`.

These four choices were confirmed with the stakeholder before implementation.

---

## 2. How Sonatype's API drives the design

Everything the integration does is grounded in the IQ Server v2 REST API
(HTTP Basic auth with a username + user token):

| Need | Endpoint | Notes |
| --- | --- | --- |
| Organizations | `GET /api/v2/organizations` | Returns the whole collection in one call |
| Applications | `GET /api/v2/applications` | Whole collection; each app carries `organizationId` |
| Latest report per stage | `GET /api/v2/reports/applications/{internalId}` | Returns a URL summary per stage, incl. `reportDataUrl` |
| Component-level violations + counts | `GET {reportDataUrl}/policy` | The policy report: `components[].violations[]` with `policyThreatLevel` / `policyThreatCategory` |
| Components, licenses + CVEs | `GET {reportDataUrl}/raw` | The raw report: `components[]` with `licenseData` and `securityData.securityIssues[]` (CVE reference + CVSS) |
| Recommended fix versions | `POST /api/v2/components/remediation/application/{internalId}?stageId={stage}` | Returns `remediation.versionChanges[]` (e.g. `next-no-violations`) |

Policy violations, license/CVE data, and fix recommendations live in three
different endpoints, so we source each deliberately: the **policy** report drives
reports and violations; the **raw** report drives components and CVEs; and the
**remediation** API supplies each vulnerable component's recommended upgrade.

A subtle but important detail: the report-summary endpoint returns URLs but not
a bare `reportId` or severity counts. We recover the `reportId` from the trailing
segment of `reportDataUrl`, and we compute severity counts ourselves by walking
the policy report's components. Fetching the policy report via the
server-provided `reportDataUrl` (rather than reconstructing the path) sidesteps
the public-vs-internal-ID ambiguity that trips up many IQ scripts.

For live updates, IQ sends **webhooks** (configured in the admin UI) with an
`X-Nexus-Webhook-Id` header identifying the event type and an
`X-Nexus-Webhook-Signature` (HMAC-SHA1 of the raw body) for verification. The
**Application Evaluation** event even carries the application, stage, `reportId`
and severity counts — everything we need to refresh a report and its violations
from a single event.

---

## 3. Development, step by step

The build follows Ocean's documented development flow. Each step below names the
files we created and why.

### Step 1 — Scaffold

We used the standard Ocean layout (matching current first-party integrations
like Jira): a package directory for the client, a `webhook_processors/`
directory, and top-level `main.py`, `integration.py`, `initialize_client.py`,
`kinds.py`, `utils.py`, plus the `.port/` configuration folder.

### Step 2 — Core logic

**`kinds.py`** defines the four resource kinds as a `StrEnum`
(`organization`, `application`, `report`, `policyViolation`) so the same strings
are reused everywhere and typos become import errors.

**`utils.py`** holds pure, I/O-free helpers — the threat-level→severity mapping,
the identifier builders, a component display-name builder, `reportId` extraction,
and relative→absolute URL conversion. Keeping these pure makes them trivial to
unit test.

**`sonatype/client.py`** is the heart of the integration. It:

- Reuses Ocean's shared `http_async_client` (so we inherit connection pooling,
  retries and backoff) and sets HTTP Basic auth from the config.
- Centralizes all requests in `_send_api_request`, which treats `404` as
  "no data" — important because IQ returns 404 for applications that have never
  been scanned.
- Exposes batched async generators `get_organizations()` and
  `get_applications()` (the endpoints return everything at once, so we chunk into
  batches of 100 to keep Port's bulk-upsert calls sensible), plus single-item
  fetches used by webhooks.
- Provides the aggregation layer: `get_application_scan_data()` fetches an
  application's report summaries, then concurrently (bounded by a semaphore)
  fetches each policy report and builds both the enriched **report** entity
  (with computed severity counts) and the flattened **policy violation**
  entities. A parallel `get_application_component_data()` reads each stage's
  **raw** report to build **component** and deduplicated **vulnerability (CVE)**
  entities, and — when enabled — calls the Component Remediation API for each
  vulnerable component to attach the recommended upgrade version. Computed
  fields are namespaced with a `__` prefix so the mapping file stays readable.

**`initialize_client.py`** wraps client creation in `lru_cache` so a single
client instance is shared across resyncs and webhook handling.

### Step 3 — Configure the integration spec

**`.port/spec.yaml`** declares the integration to Port: title, the `Security`
section, the four supported kinds, and the configuration parameters
(`iqServerUrl`, `iqUsername`, `iqUserToken`, plus optional `webhookSecret` and
`appHost`), with secrets marked `sensitive`.

### Step 4 — Integration configuration and custom selectors

**`integration.py`** subclasses `PortAppConfig` and adds a `ReportSelector`
with an optional `stages` list, so an operator can restrict syncing to specific
lifecycle stages (e.g. only `release`) for both reports and policy violations.

### Step 5 — Resource mappings

**`.port/resources/blueprints.json`** defines the four blueprints (properties,
severity enum with colors, and the up-the-chain relations).
**`.port/resources/port-app-config.yaml`** maps each Sonatype record to its
blueprint with JQ expressions — including the derived `__`-prefixed fields for
reports and violations.

### Step 6 — Resyncs and live events

**`main.py`** registers one `@ocean.on_resync` handler per kind. Organizations
and applications stream straight through; reports and violations fan out over
applications and apply the optional stage filter.

**`webhook_processors/`** implements real-time updates, one processor per kind,
all POSTing to `/webhook`:

- `report` and `policyViolation` processors react to **Application Evaluation**
  events, fetching just the single changed report's policy data.
- `application` and `organization` processors react to **Policy Management**
  events (and application evaluations) to refresh metadata.
- A shared base (`_base.py`) verifies the `X-Nexus-Webhook-Signature`
  HMAC-SHA1 against `webhookSecret` when one is configured, using the raw request
  body the framework preserves for exactly this purpose.

Because IQ has no create-webhook API, `on_start()` doesn't try to create one; it
logs the exact URL to register and the setup docs walk the admin through it.

### Step 7 — Documentation

`README.md` (setup and reference) and this `WALKTHROUGH.md` (design and process).

### Step 8 — Testing and publishing

`tests/` covers the pure helpers, the client's aggregation/counting logic, and
the webhook processors' routing, validation and auth. See sections 4 and 5.

---

## 4. Testing

The suite (run with `pytest`) is organized as:

- **`test_utils.py`** — the severity buckets at every boundary, identifier
  builders, display-name fallbacks, `reportId` extraction and URL joining.
- **`test_client.py`** — feeds a representative policy report through
  `_build_report_entity` and `_build_violation_entities` and asserts the exact
  critical/severe/moderate counts, identifiers, relations and severities, plus
  the empty-report edge case.
- **`test_webhook_processors.py`** — that each processor only claims the events
  it should, that payload validation is correct, that `handle_event` fetches and
  returns the right entities, and that signature auth accepts/rejects correctly.

All 26 tests pass, and every module imports cleanly against the real
`port-ocean` framework.

---

## 5. Publishing it as an official integration

To submit upstream:

1. Add a Towncrier news fragment under `changelog/` and set the version in
   `pyproject.toml` (starts at `0.1.0`).
2. Ensure `ruff`, `black`, `mypy` and `pytest` all pass.
3. Open a pull request against
   [`port-labs/ocean`](https://github.com/port-labs/ocean) adding this directory
   under `integrations/`.
4. Port reviews, then builds and publishes the integration image and lists it in
   the Integrations Library.

---

## 6. Customer setup and implementation

A customer adopting this integration goes through roughly this path:

**A. Create a Sonatype service account and token.** In IQ Server, create (or
choose) a user with view access to the relevant organizations/applications and
generate a **user token** for it (Account menu → User Tokens).

**B. Install the integration.** From Port's Integrations Library, add "Sonatype
Lifecycle" and provide `iqServerUrl`, `iqUsername` and `iqUserToken`. (Self-host
via Helm/Docker is also supported using the same config as environment
variables.) On first run it performs a full sync and the four blueprints appear,
populated.

**C. Turn on real-time updates.** In IQ Server, **System Preferences → Webhooks
→ New Webhook**, point it at `<appHost>/integration/webhook`, set a Secret Key
(and mirror it in `webhookSecret`), and enable the **Application Evaluation** and
**Policy Management** event types. From then on, every scan updates the catalog
within seconds.

**D. Build on top of it.** With the data in Port, customers typically:

- Add a scorecard such as "no critical policy violations on the release-stage
  report" and track it across services.
- Relate `sonatypeApplication` to their existing `service` blueprint (by
  matching `publicId`) so Sonatype risk shows directly on each service page.
- Create automations — e.g. open a ticket or notify a channel when a new
  critical violation appears — driven by the real-time webhook updates.

---

## 7. Known limitations and future enhancements

- **Webhook registration is manual** because IQ Server has no create-webhook
  REST API. This is documented rather than faked.
- **Live events don't prune fixed violations immediately.** An Application
  Evaluation event tells us the *current* violations but not which historical
  ones were resolved, so fixed violations are cleaned up on the next scheduled
  resync rather than instantly. Increasing resync frequency narrows the window.
- **Possible future kinds/enhancements:** waivers as first-class entities;
  SBOM export per application; enriching applications with source-control links
  from IQ's Source Control API; and a global (cross-report) view of a component
  version rather than one component entity per stage report.
