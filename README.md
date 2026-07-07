# Sonatype Lifecycle

A [Port Ocean](https://ocean.port.io/) integration that ingests software
composition data from **Sonatype Lifecycle (Nexus IQ Server)** into your Port
software catalog.

It maps your Sonatype hierarchy — organizations, applications, their latest
Application Composition Reports per lifecycle stage, and the individual policy
violations found in those reports — into Port entities, and keeps them up to
date both on a schedule and in real time via Sonatype webhooks.

## What gets synced

| Kind | Blueprint | Source in Sonatype IQ |
| --- | --- | --- |
| `organization` | `sonatypeOrganization` | `GET /api/v2/organizations` |
| `application` | `sonatypeApplication` | `GET /api/v2/applications` |
| `report` | `sonatypeReport` | `GET /api/v2/reports/applications/{id}` (one entity per stage) |
| `policyViolation` | `sonatypePolicyViolation` | `{reportDataUrl}/policy` |
| `component` | `sonatypeComponent` | `{reportDataUrl}/raw` (+ Component Remediation API for fix versions) |
| `vulnerability` | `sonatypeVulnerability` | `{reportDataUrl}/raw` → `securityData.securityIssues` (deduped CVEs) |

Relations: `policyViolation → report → application → organization`, and
`component → application/report` with `component → vulnerability` (many).
Components carry Sonatype's **recommended upgrade version** (`recommendedVersion`)
when remediation lookups are enabled.

To connect Sonatype applications to your existing Port `service` blueprint, see
[`.port/resources/examples/link-applications-to-services.yaml`](./.port/resources/examples/link-applications-to-services.yaml).

## Prerequisites

- A running Sonatype IQ Server (self-hosted) or a Sonatype Cloud tenant.
- A user account (ideally a dedicated service account) with permission to view
  the organizations and applications you want to sync. A **user token** for that
  account is recommended over a password
  ([how to create one](https://help.sonatype.com/en/user-tokens.html)).
- A Port account and a Port API client ID/secret.

## Configuration

| Parameter | Required | Description |
| --- | --- | --- |
| `iqServerUrl` | ✅ | Base URL of IQ Server, e.g. `https://iq.example.com` |
| `iqUsername` | ✅ | Username / service account |
| `iqUserToken` | ✅ | User token (or password) for that account |
| `webhookSecret` | ➖ | Shared secret matching the IQ webhook "Secret Key"; enables HMAC-SHA1 verification of live events |
| `appHost` | ➖ | Public base URL of this integration; used to print the exact webhook target to register |

## Running locally

```bash
# From the integration directory
poetry install

# Provide configuration via environment variables
export OCEAN__PORT__CLIENT_ID="<your-port-client-id>"
export OCEAN__PORT__CLIENT_SECRET="<your-port-client-secret>"
export OCEAN__INTEGRATION__CONFIG__IQ_SERVER_URL="https://iq.example.com"
export OCEAN__INTEGRATION__CONFIG__IQ_USERNAME="svc-port"
export OCEAN__INTEGRATION__CONFIG__IQ_USER_TOKEN="<user-token>"

# Run a one-off sync
ocean sail
```

## Enabling real-time updates (webhooks)

Sonatype IQ Server does not expose a REST endpoint for creating webhooks, so the
webhook is registered once, manually, in the IQ admin UI:

1. In IQ Server go to **System Preferences → Webhooks → New Webhook**.
2. Set the **URL** to `<appHost>/integration/webhook`.
3. (Recommended) Set a **Secret Key** and put the same value in the
   `webhookSecret` configuration so events are verified.
4. Enable the **Application Evaluation** and **Policy Management** event types.

Application Evaluation events refresh the affected report and its policy
violations; Policy Management events refresh the affected application or
organization.

## Development

```bash
poetry install
poetry run pytest            # run the test suite
poetry run ruff check .      # lint
poetry run mypy .            # type-check
```

See [`WALKTHROUGH.md`](./WALKTHROUGH.md) for a full description of how the
integration was built, step by step.
