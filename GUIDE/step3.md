# Step 3 — Script Include and Business Rules: outbound CR notifications

When a Change Request is created or its state changes, ServiceNow must notify GitHub by calling the `repository_dispatch` API. This is done with one Script Include (reusable HTTP client) and two Business Rules (triggers).

---

## 3.1 Create the Script Include `GitHubRepositoryDispatch`

1. Go to **System Definition → Script Includes**.
2. Click **New**.
3. Fill in:

| Field | Value |
|-------|-------|
| Name | `GitHubRepositoryDispatch` |
| Client callable | **false** (unchecked) |
| Description | Sends repository_dispatch events to GitHub via RESTMessageV2 |

4. Paste the script below into the **Script** editor.
5. Click **Submit**.

```javascript
var GitHubRepositoryDispatch = Class.create();
GitHubRepositoryDispatch.prototype = {
    initialize: function() {
        this.config  = this._loadConfig();
        this.token   = this.config.token || '';
        this.owner   = this.config.owner || '';
        this.repo    = this.config.repo  || '';
        this.baseUrl = 'https://api.github.com/repos/' + this.owner + '/' + this.repo + '/dispatches';
    },

    _loadConfig: function() {
        var raw = gs.getProperty('github.dispatch.config', '{}');
        try {
            return JSON.parse(raw);
        } catch (e) {
            gs.error('GitHubRepositoryDispatch: Invalid JSON in github.dispatch.config');
            return {};
        }
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
            rm.setRequestBody(JSON.stringify({
                event_type: eventType,
                client_payload: clientPayload || {}
            }));
            var response = rm.execute();
            var status   = response.getStatusCode();
            var ok       = (status === 204); // GitHub returns 204 No Content on success
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

---

## 3.2 Create Business Rule: `Notify GitHub on CR created`

Fires when a Change Request is inserted. Sends a `servicenow-cr-update` event to GitHub.

1. Go to **System Definition → Business Rules**.
2. Click **New**.
3. Fill in the header:

| Field | Value |
|-------|-------|
| Name | `Notify GitHub on CR created` |
| Table | `Change Request [change_request]` |
| Active | checked |
| Advanced | checked |

4. In the **When to run** tab:

| Field | Value |
|-------|-------|
| When | `after` |
| Insert | checked |
| Update | unchecked |
| Delete | unchecked |

5. In the **Advanced** tab, paste this script:

```javascript
(function executeRule(current, previous) {

    // Get the GitHub issue number directly from the CR (populated by the GitHub workflow).
    var issueNumber = current.u_github_issue_number.toString();

    // Fallback: if the CR was created manually and linked to a Case, check the parent Case.
    if (!issueNumber && current.parent && current.parent.isValidRecord()) {
        var parentCase = new GlideRecord('sn_customerservice_case');
        if (parentCase.get(current.parent.sys_id.toString())) {
            issueNumber = parentCase.u_github_issue_number.toString();
        }
    }
    if (!issueNumber) return;

    // Count how many CRs exist for the same parent Case (for informational purposes).
    var sibling = new GlideRecord('change_request');
    sibling.addQuery('parent', current.parent.toString());
    sibling.query();
    var total = sibling.getRowCount();

    var dispatcher = new GitHubRepositoryDispatch();
    dispatcher.send('servicenow-cr-update', {
        github_issue_number: issueNumber,
        cr_number:           current.number.toString(),
        cr_sys_id:           current.sys_id.toString(),
        cr_state:            current.state.getDisplayValue(),
        case_sys_id:         current.parent.toString(),
        action:              'created',
        total_crs:           total,
        is_first_cr:         (total === 1)
    });

})(current, previous);
```

6. Click **Submit**.

---

## 3.3 Create Business Rule: `Notify GitHub on CR state change`

Fires when a Change Request's state field changes.

1. Go to **System Definition → Business Rules**.
2. Click **New**.
3. Fill in the header:

| Field | Value |
|-------|-------|
| Name | `Notify GitHub on CR state change` |
| Table | `Change Request [change_request]` |
| Active | checked |
| Advanced | checked |

4. In the **When to run** tab:

| Field | Value |
|-------|-------|
| When | `after` |
| Insert | unchecked |
| Update | checked |
| Delete | unchecked |
| Filter Conditions | **State** `changes` |

5. In the **Advanced** tab, paste this script:

```javascript
(function executeRule(current, previous) {

    var issueNumber = current.u_github_issue_number.toString();

    if (!issueNumber && current.parent && current.parent.isValidRecord()) {
        var parentCase = new GlideRecord('sn_customerservice_case');
        if (parentCase.get(current.parent.sys_id.toString())) {
            issueNumber = parentCase.u_github_issue_number.toString();
        }
    }
    if (!issueNumber) return;

    var dispatcher = new GitHubRepositoryDispatch();
    dispatcher.send('servicenow-cr-update', {
        github_issue_number: issueNumber,
        cr_number:           current.number.toString(),
        cr_sys_id:           current.sys_id.toString(),
        cr_state:            current.state.getDisplayValue(),
        cr_environment:      current.u_environment ? current.u_environment.getDisplayValue() : '',
        case_sys_id:         current.parent.toString(),
        action:              'state_changed',
        previous_state:      previous.state.getDisplayValue()
    });

})(current, previous);
```

6. Click **Submit**.

---

## 3.4 Quick connectivity test

Run this from **System Definition → Scripts - Background** to confirm the Script Include can reach GitHub:

```javascript
var d = new GitHubRepositoryDispatch();
var result = d.send('servicenow-cr-update', {
    github_issue_number: '1',
    action:              'state_changed',
    cr_number:           'CHG0000001',
    cr_sys_id:           gs.generateGUID(),
    cr_state:            'New',
    previous_state:      'Draft',
    cr_environment:      'Development',
    case_sys_id:         gs.generateGUID()
});
gs.info(JSON.stringify(result));
```

Expected result: `{"ok":true,"status":204,"body":""}`.

If you see:
- **401 / 403** — token is invalid, expired, or missing the required scope.
- **404** — `owner` or `repo` is wrong in `github.dispatch.config`.
- **422** — `event_type` is missing or `client_payload` is not a JSON object.

---

Next: [step4.md](step4.md) — End-to-end testing
