## Step 1 — Configure ServiceNow to call GitHub (repository_dispatch)

Goal: ServiceNow must be able to send this request to GitHub:

- `POST https://api.github.com/repos/<OWNER>/<REPO>/dispatches`
- with JSON body `{ "event_type": "...", "client_payload": {...} }`

This repo listens for (at least) these `event_type` values:

- `servicenow-cr-update` (CR created / CR state changed → issue comments)

---

### 1) Create the system property `github.dispatch.config`

1. Go to **System Properties → All Properties** (or use **System Definition → System Properties** depending on your UI).
2. Click **New**.
3. Set:
   - **Name**: `github.dispatch.config`
   - **Type**: String (or Password/Encrypted if your org uses that)
   - **Value**: JSON (example below)

Use this JSON shape (your example is correct):

```json
{
  "token": "github_pat_xxxxx",
  "owner": "123",
  "repo": "234"
}
```

Notes:

- **token**: GitHub PAT that can call `repository_dispatch` on the repo.
- **owner/repo**: GitHub org/user + repo name.

---

### 2) Create Script Include `GitHubRepositoryDispatch`

1. Go to **System Definition → Script Includes**.
2. Click **New**.
3. Fill:
   - **Name**: `GitHubRepositoryDispatch`
   - **Client callable**: **false**
   - **Accessible from**: (follow your org policy; usually “All application scopes” is not needed)
4. Paste your Script Include (recommended with 2 small additions):
   - set a timeout
   - validate `eventType` exists

Use this version (based on yours):

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
        try {
            return JSON.parse(raw);
        } catch (e) {
            gs.error('GitHubRepositoryDispatch: Invalid JSON in github.dispatch.config. Error: ' + e.message);
            return {};
        }
    },

    send: function(eventType, clientPayload) {
        if (!eventType) {
            return { ok: false, status: 0, body: 'missing_event_type' };
        }
        if (!this.token || !this.owner || !this.repo) {
            gs.error('GitHubRepositoryDispatch: Missing token, owner, or repo in github.dispatch.config');
            return { ok: false, status: 0, body: 'configuration_error' };
        }

        try {
            var rm = new sn_ws.RESTMessageV2();
            rm.setHttpMethod('POST');
            rm.setEndpoint(this.baseUrl);
            rm.setHttpTimeout(30000); // 30 seconds
            rm.setRequestHeader('Accept', 'application/vnd.github+json');
            rm.setRequestHeader('Authorization', 'Bearer ' + this.token);
            rm.setRequestHeader('X-GitHub-Api-Version', '2022-11-28');
            rm.setRequestHeader('Content-Type', 'application/json');

            rm.setRequestBody(JSON.stringify({
                event_type: eventType,
                client_payload: clientPayload || {}
            }));

            var response = rm.execute();
            var status = response.getStatusCode();
            var responseBody = response.haveError() ? response.getErrorMessage() : response.getBody();
            var ok = (status === 204); // GitHub returns 204 No Content on success

            if (!ok) {
                gs.error('GitHubRepositoryDispatch failed. Status=' + status + ' Response=' + responseBody);
            }
            return { ok: ok, status: status, body: responseBody };
        } catch (ex) {
            gs.error('GitHubRepositoryDispatch exception: ' + ex.message);
            return { ok: false, status: 0, body: ex.message };
        }
    },

    type: 'GitHubRepositoryDispatch'
};
```

---

### 3) Quick connectivity test (server-side)

Run this from **Scripts - Background** (or a test Script Include / fix script):

```javascript
var d = new GitHubRepositoryDispatch();
var result = d.send('servicenow-cr-update', {
  github_issue_number: 1,
  action: 'state_changed',
  cr_number: 'CHG0000001',
  cr_sys_id: gs.generateGUID(),
  cr_state: 'New',
  previous_state: 'Draft',
  cr_environment: 'Development',
  case_sys_id: gs.generateGUID()
});
gs.info(JSON.stringify(result));
```

Expected:

- `result.ok = true`
- `result.status = 204`

If you see 401/403/404, fix the PAT / owner / repo.

