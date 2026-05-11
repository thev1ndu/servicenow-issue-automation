## Step 3 — Flow Designer: dispatch when a Change Request state changes

Goal: when the **state** of a Change Request changes, call GitHub with:

- `event_type = "servicenow-cr-update"`
- `client_payload.action = "state_changed"`
- include `previous_state` and `cr_state`

GitHub will then comment on the issue with “Change Request State Updated”.

---

### 1) Create the flow

1. Go to **Flow Designer → New → Flow**
2. Name: `GitHub - CR state changed (dispatch)`
3. Trigger:
   - **Record**
   - Table: **Change Request** (`change_request`)
   - When: **Updated**
4. Condition:
   - **State** *changes* (use the Flow condition builder so it only runs on transitions)

---

### 2) Capture previous state

How to get `previous_state` depends on your ServiceNow release:

- If Flow exposes “previous value” pills for `state`, use them.
- If it does not, create a **Before Business Rule** on `change_request` that copies `state` into a custom field (example: `u_previous_state`) before update.

Then the Flow can read `u_previous_state` as the previous value.

---

### 3) Add “Run Script”

Use this (adjust field names):

```javascript
(function execute(inputs, outputs) {
  var cr = inputs.current || inputs.change_request || null;
  if (!cr) return;

  var issueNum = cr.u_github_issue_number || '';
  if (!issueNum) return;

  var payload = {
    github_issue_number: parseInt(issueNum, 10),
    action: 'state_changed',
    cr_number: (cr.number || '') + '',
    cr_sys_id: (cr.sys_id || '') + '',
    cr_state: (cr.state || '') + '',
    previous_state: (cr.u_previous_state || '') + '',
    case_sys_id: (cr.parent || '') + ''
  };

  var d = new GitHubRepositoryDispatch();
  var res = d.send('servicenow-cr-update', payload);

  outputs.ok = res.ok;
  outputs.status = res.status;
})(inputs, outputs);
```

---

### 4) Activate and test

1. Activate the flow.
2. Move a CR from one state to another.
3. In GitHub issue, confirm a comment:
   - “Change Request State Updated”
   - “State Change: <previous> → <current>”

