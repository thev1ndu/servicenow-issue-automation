# GitHub Issues ↔ ServiceNow — setup guide

This repository uses **GitHub Actions** to open ServiceNow cases from labeled issues and to post **Change Request (CR)** updates back to the matching GitHub issue. ServiceNow only needs to call GitHub’s **[repository dispatch](https://docs.github.com/en/rest/repos/repos#create-a-repository-dispatch-event)** API when a CR is created or when its state changes.

This guide uses **Flow Designer** (ServiceNow’s visual workflow builder; sometimes still referred to as “workflow” in the UI) and **server-side REST scripting** with **`sn_ws.RESTMessageV2`**. That is the supported way to send **outbound** HTTPS from ServiceNow.

**Important platform note:** In ServiceNow, **Scripted REST API** is mainly for **inbound** APIs (external systems call *your* instance). Calling GitHub is **outbound**, so the implementation lives in a **Script Include** (or REST Message) using **`RESTMessageV2`**, which Flow Designer invokes with a **Run Script** step. If your standards require the Scripted REST *module* as the container, you can paste the same script into a Scripted REST resource and call it from Flow; the GitHub payload and headers below stay the same.

---

## What you need first

| Item | Purpose |
|------|--------|
| GitHub org/user and repository name | Build the dispatch URL |
| GitHub token | `POST /repos/{owner}/{repo}/dispatches` |
| Issue number on GitHub for each case | Stored on the case (or derived) as `github_issue_number` in the payload |
| ServiceNow admin or `sn_flow_designer` + script access | Create Flow + Script Include |

### GitHub token

1. In GitHub: **Settings → Developer settings → Personal access tokens** (fine-grained or classic).
2. For a **private** repo, the token needs at least:
   - **Contents:** Read and write (repository dispatch is treated under repo content scope), or use a classic token with **`repo`** scope for private repositories.
3. For **fine-grained** tokens: select the repository and grant **Contents** read/write (and **Metadata** read).
4. Copy the token once; you will store it only in ServiceNow (credential alias or `sys_properties` — see below).

### Dispatch URL

```
https://api.github.com/repos/<OWNER>/<REPO>/dispatches
```

Example: `https://api.github.com/repos/acme-corp/AutomationChoreoSR/dispatches`

### Event types this repo listens for

| `event_type` | When to use |
|--------------|-------------|
| `servicenow-cr-update` | CR created or CR state changed (required for CR → issue comments) |
| `validation-passed` | Optional manual / automation path to (re)run case creation from GitHub; usually the issue pipeline adds the label itself |

The workflows that handle these are `.github/workflows/servicenow-inbound.yml` and `.github/workflows/servicenow-create-case.yml`.

---

## Part 1 — Script Include (outbound GitHub dispatch)

Create a **Script Include** so Flow can call one place for all GitHub `repository_dispatch` calls.

1. Navigate to **System Definition → Script Includes → New**.
2. **Name:** `GitHubRepositoryDispatch` (or your naming standard).
3. **API name:** same as name, **Client callable:** false, **Application scope:** your scoped app or Global (follow your org rules).
4. **Script:** use the template below; replace property names with yours.

### Script Include template (Glide / Rhino)

```javascript
var GitHubRepositoryDispatch = Class.create();
GitHubRepositoryDispatch.prototype = {
    initialize: function() {
        // Optional: load from System Properties [github.token], [github.owner], [github.repo]
        this.token = gs.getProperty('github.dispatch.token', '');
        this.owner = gs.getProperty('github.dispatch.owner', '');
        this.repo = gs.getProperty('github.dispatch.repo', '');
        this.baseUrl = 'https://api.github.com/repos/' + this.owner + '/' + this.repo + '/dispatches';
    },

    /**
     * @param {String} eventType - e.g. servicenow-cr-update | validation-passed
     * @param {Object} clientPayload - JSON-serializable object (GitHub client_payload)
     */
    send: function(eventType, clientPayload) {
        if (!this.token || !this.owner || !this.repo) {
            gs.error('GitHubRepositoryDispatch: missing token, owner, or repo property');
            return { ok: false, status: 0, body: 'configuration' };
        }
        var rm = new sn_ws.RESTMessageV2();
        rm.setHttpMethod('POST');
        rm.setEndpoint(this.baseUrl);
        rm.setRequestHeader('Accept', 'application/vnd.github+json');
        rm.setRequestHeader('Authorization', 'Bearer ' + this.token);
        rm.setRequestHeader('X-GitHub-Api-Version', '2022-11-28');
        rm.setRequestHeader('Content-Type', 'application/json');

        var body = {
            event_type: eventType,
            client_payload: clientPayload || {}
        };
        rm.setRequestBody(JSON.stringify(body));

        var response = rm.execute();
        var status = response.getStatusCode();
        var responseBody = response.haveError() ? response.getErrorMessage() : response.getBody();
        var ok = (status === 204);
        if (!ok) {
            gs.error('GitHubRepositoryDispatch failed: status=' + status + ' body=' + responseBody);
        }
        return { ok: ok, status: status, body: responseBody };
    },

    type: 'GitHubRepositoryDispatch'
};
```

5. **System Properties** (recommended): **System Definition → System Properties → New** (or set in your scoped app config):

| Name | Value |
|------|--------|
| `github.dispatch.token` | GitHub PAT (mark as **masked** if you use a wrapper property) |
| `github.dispatch.owner` | GitHub org or username |
| `github.dispatch.repo` | Repository name |

**Security:** Prefer **Credentials** and **Connection & Credential Aliases** with a **REST** credential type, and read the secret in script only if your security team allows it; otherwise keep the token in an encrypted system property or Edge Encryption–enabled storage per your policy.

---

## Part 2 — Payload contracts (must match GitHub Actions)

### A. `servicenow-cr-update` (`event_type`)

GitHub workflow reads `client_payload` fields (see `servicenow-inbound.yml`).

| Field | Required | Description |
|-------|----------|-------------|
| `github_issue_number` | **Yes** | GitHub issue number (integer or string) |
| `action` | **Yes** for comments | `created` or `state_changed` |
| `cr_number` | For display | e.g. `CHG0001234` |
| `cr_sys_id` | Recommended | Change Request `sys_id` (for links) |
| `cr_state` | Recommended | Display / current state label |
| `cr_environment` | Optional | Defaults to `Not specified` in GitHub |
| `case_sys_id` | Recommended | Parent case `sys_id` (GitHub queries linked CRs) |
| `previous_state` | For `state_changed` | Prior state label |

**Example — CR created**

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

**Example — CR state changed**

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

### B. `validation-passed` (optional)

Used only if you intentionally trigger GitHub’s `servicenow-inbound.yml` → `servicenow-create-case.yml` path from ServiceNow (for example a manual “retry case creation” flow).

| Field | Required | Description |
|-------|----------|-------------|
| `issue_number` | **Yes** | GitHub issue number |
| `catalog_item` | Optional | Default `General Requests` |
| `case_type` | Optional | Default `Service Request` |
| `sr_type` | Optional | Free text |

---

## Part 3 — Flow Designer (“Workflow Builder”) setup

Use **Flow Designer** so you do not maintain legacy **Workflow** (`wf_workflow`) XML unless required.

### Flow 1 — Notify GitHub when a Change Request is **inserted**

1. **Flow Designer → New → Flow**.
2. **Name:** e.g. `GitHub - CR created dispatch`.
3. **Run as:** System (if the Script Include reads system properties with a service account token).
4. **Trigger:** **Record** → **Change Request** (`change_request`) → **Created** (or **Inserted** depending on version wording).
5. Add a **Run Script** action (or **Run Script** in a subflow) **after** trigger, with script similar to:

```javascript
(function execute(inputs, outputs) {
    // Map from trigger record — adjust field names to your case/CR model
    var cr = inputs.change_request; // Flow passes current record in your SN version; if not, use a "Get Record" step first

    var issueNum = cr.u_github_issue_number; // replace with your field that stores GitHub issue #
    if (!issueNum) {
        return;
    }

    var dispatch = new GitHubRepositoryDispatch();
    var payload = {
        github_issue_number: parseInt(issueNum, 10),
        action: 'created',
        cr_number: cr.number + '',
        cr_sys_id: cr.sys_id + '',
        cr_state: (cr.state + '') || '',
        cr_environment: cr.u_environment || 'Not specified', // replace with your field
        case_sys_id: (cr.parent + '') || '' // parent case sys_id if CR.parent points to case task
    };

    var result = dispatch.send('servicenow-cr-update', payload);
    outputs.http_status = result.status;
    outputs.ok = result.ok;
})(inputs, outputs);
```

**Note:** In Flow Designer, the **trigger record** is often exposed as `inputs.current` or via a **Get Record ID from Trigger** pattern depending on release. Use **Data Pill** picker to bind `change_request` fields instead of guessing variable names. Replace `u_github_issue_number` with the field your integration uses to store the GitHub issue number (populated when the case is created from GitHub or from your catalog).

6. **Save** and **Activate** the flow.

### Flow 2 — Notify GitHub when Change Request **state** changes

1. **New Flow** → **Name:** `GitHub - CR state changed dispatch`.
2. **Trigger:** **Record** → **Change Request** → **Updated**.
3. Add a **Condition** (Flow logic): **State** (or `state`) **is changed** — use the condition builder so the flow only runs on state transitions.
4. **Run Script** similar to Flow 1, but:

```javascript
(function execute(inputs, outputs) {
    var cr = inputs.change_request; // bind via pills
    var issueNum = cr.u_github_issue_number;
    if (!issueNum) return;

    var dispatch = new GitHubRepositoryDispatch();
    var payload = {
        github_issue_number: parseInt(issueNum, 10),
        action: 'state_changed',
        cr_number: cr.number + '',
        cr_sys_id: cr.sys_id + '',
        cr_state: (cr.state + '') || '',
        previous_state: (inputs.previous_state || '') + '', // from "Get Change" before update or audit — see note below
        case_sys_id: (cr.parent + '') || ''
    };
    outputs.result = dispatch.send('servicenow-cr-update', payload);
})(inputs, outputs);
```

**Previous state:** Flow **Record Updated** triggers often expose **previous values** in the **Trigger** payload or via a **Get Record** step on `sys_audit` / **Business Rule**-style scratchpad. If your instance does not expose `previous_state` easily, add a **Before Business Rule** on `change_request` that copies `current.state` to `u_previous_state` on the record before update, then read `u_previous_state` from the **current** record in the Flow (adjust design to your standards).

### Testing the Flow

1. Open **Flow Designer → Flow executions** (or **Monitor**) and run a test CR with a known `github_issue_number`.
2. In GitHub: **Actions** tab → confirm a run of **ServiceNow inbound** for `servicenow-cr-update`.
3. On the issue: confirm a new comment from the workflow.

---

## Part 4 — Scripted REST API (all-in-one resource)

Use this when you want the GitHub call to live **only** in **Scripted REST** (plus Flow). The resource accepts JSON from Flow (or another server-side caller), then uses **`sn_ws.RESTMessageV2`** to call GitHub (outbound). Scripted REST here is the **container**; GitHub is still reached via **`RESTMessageV2`** (platform standard).

### 4.1 Create the API

1. Open your **scoped application** (recommended) or stay in Global per policy.
2. **Scripted REST APIs → New**  
   - **API ID:** `github_repository_dispatch` (example)  
   - **API name:** human-readable label  
   - **Active:** true  
3. **Save**, then **Resources → New**:
   - **HTTP method:** `POST`
   - **Relative path:** `/repository_dispatch` (full path becomes `/api/<namespace>/<api_id>/repository_dispatch` — exact prefix depends on scope; use **Preview** in the form to copy the full URL).
   - **Requires authentication:** true (so only authenticated ServiceNow users/flows call it).
   - **Requires ACL authorization:** true — create an **ACL** on this API resource limited to **admin** and your **integration user** role.

### 4.2 Resource script (paste into the Scripted REST resource)

Request body from Flow should be:

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": 42,
    "action": "created",
    "cr_number": "CHG0001001",
    "cr_sys_id": "...",
    "cr_state": "New",
    "cr_environment": "Production",
    "case_sys_id": "...",
    "previous_state": ""
  }
}
```

Resource script:

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {
    var body = request.body.data;
    if (!body || !body.event_type) {
        response.setStatus(400);
        response.setBody({ error: 'event_type required' });
        return;
    }

    var token = gs.getProperty('github.dispatch.token', '');
    var owner = gs.getProperty('github.dispatch.owner', '');
    var repo = gs.getProperty('github.dispatch.repo', '');
    if (!token || !owner || !repo) {
        response.setStatus(500);
        response.setBody({ error: 'GitHub properties not configured' });
        return;
    }

    var endpoint = 'https://api.github.com/repos/' + owner + '/' + repo + '/dispatches';
    var rm = new sn_ws.RESTMessageV2();
    rm.setHttpMethod('POST');
    rm.setEndpoint(endpoint);
    rm.setRequestHeader('Accept', 'application/vnd.github+json');
    rm.setRequestHeader('Authorization', 'Bearer ' + token);
    rm.setRequestHeader('X-GitHub-Api-Version', '2022-11-28');
    rm.setRequestHeader('Content-Type', 'application/json');
    rm.setRequestBody(JSON.stringify({
        event_type: body.event_type,
        client_payload: body.client_payload || {}
    }));

    var res = rm.execute();
    var code = res.getStatusCode();
    var ok = (code === 204);
    response.setStatus(ok ? 200 : 502);
    response.setBody({
        ok: ok,
        github_status: code,
        message: res.haveError() ? res.getErrorMessage() : res.getBody()
    });
})(request, response);
```

### 4.3 Call Scripted REST from Flow Designer

1. In your **Change Request** flow, add action **Send REST Message** / **REST** (wording varies by release) — or use **Run Script** with `RESTMessageV2` pointed at **your instance** Scripted REST URL (same pattern as GitHub but with Basic / OAuth for the instance).
2. **Method:** `POST`
3. **Relative path / URL:** paste the **full Scripted REST URL** from the API record (e.g. `https://<instance>.service-now.com/api/x_yourapp/github_repository_dispatch/repository_dispatch`).
4. **Authentication:** user whose session can pass the ACL (often **Run as** system with **Connection** using **Credential** for the integration user).
5. **Request body:** build JSON with **Data pills** from the trigger record (`github_issue_number`, `cr_number`, etc.) matching the structure in §4.2.

**Tip:** If your Flow version has **Run Script** only, you can skip Scripted REST and call `GitHubRepositoryDispatch` from Part 1 directly; both approaches end with the same GitHub API call.

---

## Part 5 — GitHub repository checklist (for your admins)

Secrets used by Actions (names may vary in your fork):

- `SERVICENOW_URL`, `SERVICENOW_UI_URL`, `SERVICENOW_USERNAME`, `SERVICENOW_PASSWORD` — used by workflows to read CRs and post issue comments.
- Workflows: `issue-servicenow.yml`, `servicenow-create-case.yml`, `servicenow-inbound.yml`.

No GitHub configuration is required beyond the token and dispatch URL described above.

---

## Troubleshooting

| Symptom | Check |
|--------|--------|
| HTTP 404 from GitHub | `OWNER` / `REPO` wrong or token cannot see repo |
| HTTP 401 / 403 | Token scopes; fine-grained token must include **Contents** on that repo |
| HTTP 422 | Body must include `event_type`; `client_payload` must be a JSON object (can be empty for some events, not for ours) |
| GitHub Actions never runs | Repo **Settings → Actions** enabled; `event_type` string must match **exactly** `servicenow-cr-update` or `validation-passed` |
| No issue comment | `github_issue_number` missing or wrong; `action` not `created` / `state_changed` for CR flow |
| Duplicate comments | Add Flow **Condition** or BR debounce so the same transition does not fire twice |

---

## Summary

1. Store **GitHub PAT** and **owner/repo** in ServiceNow (properties or credentials).
2. Add **Script Include** `GitHubRepositoryDispatch` using **`sn_ws.RESTMessageV2`** to `POST .../dispatches`.
3. Build **Flow Designer** flows on **Change Request** insert and state change that **Run Script** and pass **`servicenow-cr-update`** payloads matching the tables in this README.
4. Keep **`github_issue_number`** (or your field mapped to it) populated on the CR or parent case so ServiceNow knows which issue to update.

That completes the ServiceNow side using **Flow Designer** and **scripted outbound REST** (`RESTMessageV2`) aligned with how GitHub Actions in this repository expect to be triggered.
