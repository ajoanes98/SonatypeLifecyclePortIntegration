# Self-service actions

These two JSON files are exported, ready-to-use definitions of this
integration's self-service actions. They are **optional** and not created
automatically — Ocean's resource scaffolding (`.port/resources/blueprints.json`,
`.port/resources/port-app-config.yaml`) only provisions blueprints and data
mapping, not actions, so bringing these in is a deliberate opt-in step you
take if/when you want them.

## Adding an action to your Port organization

1. Go to the **Self-service** page in Port.
2. Click **+ New Action** → **{...} Edit JSON**.
3. Paste the contents of the JSON file you want (see placeholders below —
   fill those in first, or edit them afterwards in the UI).
4. Click **Create**.

Repeat for the other file if you want both. There's no dependency between
them — take either one independently.

## Prerequisites

Both actions assume the blueprint/mapping setup this integration provisions
is already in place, plus two extra pieces that live on top of it:

- **`sonatypeApplication.githubRepository`** relation and **`sonatypeComponent.repositoryIdentifier`**
  mirror property (`application.githubRepository.$identifier`) — these are
  what let `upgrade_component_to_recommended_version` auto-select the target
  repo. They're populated from the `sourceControl` kind, so a repository is
  only pre-filled for applications that actually have Source Control
  Management configured in IQ Server.
- The **GitHub Ocean/GitHub App integration** installed, with a
  `githubRepository` blueprint present — `upgrade_component_to_recommended_version`'s
  `target_repository` input targets that blueprint. Without it installed,
  the action will still import, but the input will have nothing to pick
  from.

## Placeholders you need to fill in before these are usable

### `request_policy_violation_waiver.json`
- `REPLACE_WITH_YOUR_IQ_SERVER_BASE_URL` in `invocationMethod.url` — your
  actual IQ Server host.
- IQ's Policy Waiver Request API uses basic auth. Add an `Authorization`
  header to `invocationMethod.headers` sourced from a Port secret (Settings
  → Credentials → Secrets), e.g. `"Authorization": "Basic {{ .secrets._IQ_BASIC_AUTH }}"`.
- If IQ Server is on-prem and not reachable from Port's SaaS webhook egress,
  point the URL at a relay (e.g. an n8n workflow or small Lambda) that
  forwards to IQ instead of calling it directly.

### `upgrade_component_to_recommended_version.json`
- `<AUTOMATION_GITHUB_ORG>` / `<AUTOMATION_GITHUB_REPO>` in
  `invocationMethod.url` — the GitHub org/repo where you've placed
  `upgrade-component-version.yml` (the workflow this action dispatches;
  ships alongside this integration — see the repo root for it).
- Port secret `_GITHUB_TOKEN` — a token with `Actions: write` on that
  automation repo, so Port can dispatch the workflow.
- In the automation repo itself: a `GH_PAT` secret with `Contents` + `Pull
  requests` write access on whichever repos the upgrade PRs will target
  (the default `GITHUB_TOKEN` can't push cross-repo).

## Permissions

Neither JSON file sets custom execute/approve permissions — both import
with Port's default (organization admins can approve/execute). If you want
non-admins to request waivers with a specific approver group, or want to
restrict who can trigger the upgrade action, set that up afterwards in the
Self-service page → action → Permissions tab, or via the
[update an action's permissions API](https://docs.port.io/api-reference/update-an-actions-permissions).
This isn't exported here because permissions are org-specific by nature.
