# GitHub Issues → ServiceNow Case Automation

A **template and testing repository** for automatically creating ServiceNow cases from GitHub issues and posting Change Request updates back to GitHub.

---

## What this does

```
GitHub Issue (labeled)
        │
        ▼
  [Validation]  ← checks template, title format, required fields, label combo
        │
        ▼
  [Case Created] → POST to ServiceNow Scripted REST API → ServiceNow Case
        │
        ▼
  GitHub issue gets a comment with the case number + link

ServiceNow CR created/state changed
        │
        ▼
  ServiceNow fires repository_dispatch → GitHub
        │
        ▼
  GitHub issue gets a comment with CR number + state
```

---

## Repository structure

```
.
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── config.yml               # disables blank issues
│   │   ├── sr-generic.md            # CatalogueItem/Generic Requests
│   │   ├── sr-request-logs.md       # CatalogueItem/Request Logs
│   │   ├── sr-standard-generic.md   # CatalogueItem/Standard Generic
│   │   └── sr-information.md        # CatalogueItem/Information
│   ├── labels.yml                   # label definitions (apply once)
│   └── workflows/
│       ├── issue-servicenow.yml     # main trigger: issue opened/labeled
│       ├── servicenow-create-case.yml  # reusable: build payload + POST to SN
│       └── servicenow-inbound.yml   # handle ServiceNow → GitHub events
├── GUIDE/
│   ├── README.md                    # ServiceNow setup index
│   ├── step1.md                     # Script Include + system property
│   ├── step2.md                     # Flow: CR created → dispatch
│   ├── step3.md                     # Flow: CR state changed → dispatch
│   └── step4.md                     # End-to-end testing
├── HOWTHISWORKS.md                  # full end-to-end explanation
├── ServiceNow.md                    # what was built on the SN instance
├── LINKS.md                         # ServiceNow instance URL reference
└── samples.md                       # curl samples to test without SN
```

---

## Quick start

### Prerequisites

| Item | Where |
|------|-------|
| ServiceNow instance with CSM (Customer Service Management) | Your SN admin |
| GitHub repo admin access | To set secrets and create labels |
| ServiceNow admin or integration-user access | To create REST API + Script Include |

### Step 1 — Set GitHub secrets

In your repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|-------------|-------|
| `SERVICENOW_URL` | Full URL to the Scripted REST endpoint, e.g. `https://<instance>.service-now.com/api/<scope>/github_case/v1/case` |
| `SERVICENOW_UI_URL` | Base instance URL, e.g. `https://<instance>.service-now.com` |
| `SERVICENOW_USERNAME` | ServiceNow user with REST access |
| `SERVICENOW_PASSWORD` | Password for the above user |

### Step 2 — Configure ServiceNow

Follow the step-by-step guide in [GUIDE/step1.md](GUIDE/step1.md):

1. Create system property `github.dispatch.config` with your GitHub PAT + owner + repo
2. Create Script Include `GitHubRepositoryDispatch`
3. Create the Scripted REST API (`GitHub Case Integration`) — see [ServiceNow.md](ServiceNow.md) for the full script
4. Create Business Rules (or Flows) for CR created/state changed — see [GUIDE/step2.md](GUIDE/step2.md) and [GUIDE/step3.md](GUIDE/step3.md)

### Step 3 — Create labels in GitHub

Apply all labels from [.github/labels.yml](.github/labels.yml) to your repo. You can use:

```bash
gh label create "CatalogueItem/Generic Requests" --color e4e669 --description "Catalog: Generic Requests (full field set)"
gh label create "CatalogueItem/Request Logs"     --color e4e669 --description "Catalog: Request Logs (Priority must be Critical)"
gh label create "CatalogueItem/Standard Generic" --color e4e669 --description "Catalog: Standard Generic (reduced field set)"
gh label create "CatalogueItem/Information"      --color e4e669 --description "Catalog: Information Request"
gh label create "SRType/Normal Change"           --color 0075ca --description "SR type: Normal Change"
gh label create "SRType/Standard Change"         --color 0075ca --description "SR type: Standard Change"
gh label create "SRType/Emergency Change"        --color d93f0b --description "SR type: Emergency Change"
gh label create "validation-passed"              --color 0e8a16 --description "SR template validated — case will be created automatically"
```

### Step 4 — Customize for your project

In [.github/workflows/servicenow-create-case.yml](.github/workflows/servicenow-create-case.yml), update the two env vars at the top:

```yaml
env:
  SERVICENOW_PRODUCT: "your product name"
  SERVICENOW_PROJECT: "your project name"
```

These are sent to ServiceNow with every case.

### Step 5 — Test

Use [samples.md](samples.md) to send test `repository_dispatch` events without touching ServiceNow.

For the full label-driven pipeline, create an issue from one of the SR templates, add the required labels, and watch the workflow run.

---

## How to use (end users)

1. Open a new issue using one of the SR templates (click **New Issue** in GitHub).
2. Fill in all fields — do not leave `_No response_` in required sections.
3. Set the issue title to start with `[SR-Change]:`.
4. Add **both** required labels:
   - One `SRType/` label (e.g. `SRType/Normal Change`)
   - One `CatalogueItem/` label matching the template you used
5. The workflow validates the issue. On success it posts "✅ Template Validation Passed" and creates the ServiceNow case automatically.
6. When ServiceNow creates or updates a Change Request, a comment appears on the issue automatically.

---

## Workflow reference

| Workflow | Trigger | What it does |
|----------|---------|--------------|
| `issue-servicenow.yml` | Issue opened, edited, or labeled | Gate → Validate → Create case → Watch for CRs |
| `servicenow-create-case.yml` | `workflow_call` | Extract fields from issue body, POST payload to ServiceNow, comment result |
| `servicenow-inbound.yml` | `repository_dispatch` | Handle CR created / CR state changed events from ServiceNow |

### Required labels

| Label | Set by | Purpose |
|-------|--------|---------|
| `SRType/<type>` | Human | Identifies the change type |
| `CatalogueItem/<item>` | Human | Selects the SR form type and validation rules |
| `validation-passed` | Workflow (automatic) | Signals validation passed; triggers case creation |

### Catalog item field requirements

| Label | Required fields |
|-------|----------------|
| `CatalogueItem/Generic Requests` | Short Description, Description, Priority, Impact, Impact Description (Overall), Impact Description (Customer), Environment Details, Affected Component, Affected Services, Service Outage/Downtime, Is a maintenance window required or not, Implementation Plan, Test Plan, Monitoring Checks |
| `CatalogueItem/Request Logs` | Short Description, Description, Priority (**must be Critical**), Impact, Impact Description, Customer Project, Environment Details |
| `CatalogueItem/Standard Generic` | Short Description, Description, Priority, Impact, Environment Details |
| `CatalogueItem/Information` | Request Description, Impact, Customer Project |

---

## Further reading

- [HOWTHISWORKS.md](HOWTHISWORKS.md) — full end-to-end flow explanation
- [ServiceNow.md](ServiceNow.md) — ServiceNow instance configuration reference
- [GUIDE/](GUIDE/) — step-by-step ServiceNow setup (Script Include, Flows, testing)
- [samples.md](samples.md) — curl commands to test repository_dispatch without ServiceNow

---

## ServiceNow outbound setup (calling GitHub from ServiceNow)

The rest of this document covers how ServiceNow sends Change Request events back to GitHub. This is the **outbound** direction: ServiceNow → GitHub.

ServiceNow uses **Flow Designer** and **`sn_ws.RESTMessageV2`** (the platform-supported way to make outbound HTTPS calls). A **Script Include** (`GitHubRepositoryDispatch`) wraps the GitHub API call so all flows share one implementation.

---

### Part 1 — Script Include (outbound GitHub dispatch)

1. Navigate to **System Definition → Script Includes → New**.
2. **Name:** `GitHubRepositoryDispatch`, **Client callable:** false.
3. **Script:**

```javascript
var GitHubRepositoryDispatch = Class.create();
GitHubRepositoryDispatch.prototype = {
    initialize: function() {
        this.config = this._loadConfig();
        this.token = this.config.token || '';
        this.owner = this.config.owner || '';
        this.repo = this.config.repo || '';
        this.baseUrl = 'https://api.github.com/repos/' + this.owner + '/' + this.repo + '/dispatches';
    },

    _loadConfig: function() {
        var raw = gs.getProperty('github.dispatch.config', '{}');
        try { return JSON.parse(raw); }
        catch (e) { gs.error('GitHubRepositoryDispatch: Invalid JSON in github.dispatch.config'); return {}; }
    },

    send: function(eventType, clientPayload) {
        if (!eventType) return { ok: false, status: 0, body: 'missing_event_type' };
        if (!this.token || !this.owner || !this.repo) {
            gs.error('GitHubRepositoryDispatch: Missing token, owner, or repo in github.dispatch.config');
            return { ok: false, status: 0, body: 'configuration_error' };
        }
        try {
            var rm = new sn_ws.RESTMessageV2();
            rm.setHttpMethod('POST');
            rm.setEndpoint(this.baseUrl);
            rm.setHttpTimeout(30000);
            rm.setRequestHeader('Accept', 'application/vnd.github+json');
            rm.setRequestHeader('Authorization', 'Bearer ' + this.token);
            rm.setRequestHeader('X-GitHub-Api-Version', '2022-11-28');
            rm.setRequestHeader('Content-Type', 'application/json');
            rm.setRequestBody(JSON.stringify({ event_type: eventType, client_payload: clientPayload || {} }));
            var response = rm.execute();
            var status = response.getStatusCode();
            var ok = (status === 204);
            if (!ok) gs.error('GitHubRepositoryDispatch failed: status=' + status);
            return { ok: ok, status: status, body: response.haveError() ? response.getErrorMessage() : response.getBody() };
        } catch (ex) {
            gs.error('GitHubRepositoryDispatch exception: ' + ex.message);
            return { ok: false, status: 0, body: ex.message };
        }
    },

    type: 'GitHubRepositoryDispatch'
};
```

4. Create system property `github.dispatch.config` (type: String or Password):

```json
{
  "token": "github_pat_xxxxx",
  "owner": "your-github-org-or-user",
  "repo": "your-repo-name"
}
```

---

### Part 2 — Payload contracts

#### `servicenow-cr-update` — CR created

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": 42,
    "action": "created",
    "cr_number": "CHG0001001",
    "cr_sys_id": "a1b2c3d4e5f6789012345678901234ab",
    "cr_state": "New",
    "cr_environment": "Production",
    "case_sys_id": "b2c3d4e5f6789012345678901234abcd"
  }
}
```

#### `servicenow-cr-update` — CR state changed

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": 42,
    "action": "state_changed",
    "cr_number": "CHG0001001",
    "cr_sys_id": "a1b2c3d4e5f6789012345678901234ab",
    "cr_state": "Implement",
    "previous_state": "New",
    "case_sys_id": "b2c3d4e5f6789012345678901234abcd"
  }
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `github_issue_number` | Yes | GitHub issue number |
| `action` | Yes | `created` or `state_changed` |
| `cr_number` | Yes | e.g. `CHG0001234` |
| `cr_sys_id` | Recommended | For direct link in comment |
| `cr_state` | Recommended | Display value |
| `cr_environment` | Optional | Defaults to `Not specified` |
| `case_sys_id` | Recommended | GitHub queries linked CRs using this |
| `previous_state` | For `state_changed` | Prior state display value |

---

### Part 3 — Flow Designer setup

See [GUIDE/step2.md](GUIDE/step2.md) (CR created) and [GUIDE/step3.md](GUIDE/step3.md) (CR state changed) for click-by-click Flow Designer instructions.

**Flow 1 — CR created**
- Trigger: Change Request → Created
- Run Script: call `GitHubRepositoryDispatch.send('servicenow-cr-update', { action: 'created', ... })`

**Flow 2 — CR state changed**
- Trigger: Change Request → Updated, Condition: State changes
- Run Script: call `GitHubRepositoryDispatch.send('servicenow-cr-update', { action: 'state_changed', ... })`

---

### Troubleshooting

| Symptom | Check |
|---------|-------|
| HTTP 404 from GitHub | `owner` / `repo` wrong in `github.dispatch.config` |
| HTTP 401 / 403 | Token missing permissions; fine-grained token needs **Contents: Read and write** on the repo |
| HTTP 422 | Body must include `event_type`; `client_payload` must be a JSON object |
| GitHub Actions never runs | Repo Settings → Actions enabled; `event_type` must exactly match `servicenow-cr-update` |
| No issue comment | `github_issue_number` wrong or missing; `action` not `created`/`state_changed` |
| "All CRs" list shows only one | `case_sys_id` missing from payload, or ServiceNow credentials in GitHub secrets are wrong |
