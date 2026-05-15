# Guide: CR Approval Info Subflow → GitHub

When a Change Request moves to the **Authorize** or **CAB Approval** state, this subflow
looks up who authorized or approved it and posts a notification on the linked GitHub issue.

GitHub Actions file that receives it: `.github/workflows/sn-cr-notifier.yml`
GitHub event type sent: `servicenow-cr-update` with `action: approval_updated`

---

## What happens end to end

**When the CR moves to Authorize (`-3`):**
1. Flow B (SN CR Updated → GitHub) fires
2. Script Step 1 detects state `-3` and sets `action = 'approval_updated'`
3. The If condition calls the **SN CR Approval Lookup** subflow
4. The subflow queries `sysapproval_approver` for the approved authorizer; falls back to `u_authorized_by` / `approved_by` on the CR record
5. Script Step 2 dispatches to GitHub with `authorized_by` and `approval_type: authorized`
6. `sn-cr-notifier.yml` posts a comment showing who authorized the CR

**When the CR moves to CAB Approval (`-2`):**
Same flow path, but all approved CAB approver names are fetched and sent as `cab_approvers`.

---

## Prerequisites

- [sn-cr-notifier.md](sn-cr-notifier.md) is complete — **Flow B (SN CR Updated → GitHub)** must exist and be active
- [setup-rest-message.md](setup-rest-message.md) is complete — the `GitHub Integration` REST Message and `dispatch_cr` HTTP Method must exist

---

## Part 1 — Create the Subflow: "SN CR Approval Lookup"

### 1.1 Create the Subflow

Go to: `All > Process Automation > Flow Designer`

Click **New > Subflow**

- **Name**: `SN CR Approval Lookup`
- **Description**: `Looks up who authorized or CAB-approved a Change Request`
- **Run as**: `System User`

Click **Submit**.

---

### 1.2 Define Input Variables

Click **Inputs** at the top of the subflow canvas.

| Variable Name | Type |
|---|---|
| `cr_sys_id` | String |
| `cr_state` | String |

Click **Done**.

---

### 1.3 Define Output Variables

Click **Outputs** at the top of the subflow canvas.

| Variable Name | Type |
|---|---|
| `authorized_by` | String |
| `cab_approvers` | String |

Click **Done**.

---

### 1.4 Add Script Step — Look up approval records

Click **+** on the canvas. Select **Script**.

#### Input Variables

| Variable Name | Type | Source |
|---|---|---|
| `cr_sys_id` | String | Subflow Inputs > `cr_sys_id` |
| `cr_state` | String | Subflow Inputs > `cr_state` |

#### Script

```javascript
(function execute(inputs, outputs) {

  var crSysId = inputs.cr_sys_id + '';
  var crState = inputs.cr_state + '';

  var authorizedBy = '';
  var cabApprovers = '';

  // Query all approved approval records for this CR
  var appGr = new GlideRecord('sysapproval_approver');
  appGr.addQuery('source_table', 'change_request');
  appGr.addQuery('document_id', crSysId);
  appGr.addQuery('state', 'approved');
  appGr.orderByDesc('sys_updated_on');
  appGr.query();

  var approvers = [];
  while (appGr.next()) {
    var name = appGr.getDisplayValue('approver');
    if (name) approvers.push(name);
  }

  if (crState === '-3') {
    // Authorize state: the most-recent approver is the authorizing change manager
    authorizedBy = approvers.length > 0 ? approvers[0] : '';

    // Fallback to CR-level field if the approval record is not yet committed
    if (!authorizedBy) {
      var crGr = new GlideRecord('change_request');
      if (crGr.get(crSysId)) {
        authorizedBy = crGr.getDisplayValue('u_authorized_by') ||
                       crGr.getDisplayValue('approved_by') || '';
      }
    }

  } else if (crState === '-2') {
    // CAB Approval state: all CAB approvers
    cabApprovers = approvers.join(', ');
  }

  outputs.authorized_by = authorizedBy;
  outputs.cab_approvers  = cabApprovers;

})(inputs, outputs);
```

#### Output Variables

| Variable Name | Type |
|---|---|
| `authorized_by` | String |
| `cab_approvers` | String |

Click **Done**.

---

### 1.5 Map Subflow Outputs

Click **Outputs** on the canvas. Wire:

| Subflow Output | Source |
|---|---|
| `authorized_by` | Script Step > `authorized_by` |
| `cab_approvers` | Script Step > `cab_approvers` |

Click **Done**.

---

### 1.6 Save and Activate the Subflow

Click **Save**, then **Activate**.

---

## Part 2 — Modify Flow B: SN CR Updated → GitHub

Open **SN CR Updated → GitHub** in Flow Designer and click **Edit**.

---

### 2.1 Update Script Step 1

Open the existing **Script Step 1**. Replace only the `action` assignment block with the version below.
All other inputs, outputs, and logic remain unchanged.

**Replace this existing block:**

```javascript
// state_changed takes priority; fall back to dates_updated
var action = (dataPillState !== crState) ? 'state_changed' : 'dates_updated';
```

**With:**

```javascript
// approval_updated fires for Authorize/CAB Approval; state_changed for all other state transitions
var action;
if (dataPillState !== crState) {
  action = (crState === '-3' || crState === '-2') ? 'approval_updated' : 'state_changed';
} else {
  action = 'dates_updated';
}
```

Click **Done**.

The full updated Script Step 1 for reference:

```javascript
(function execute(inputs, outputs) {

  var issueNumber  = inputs.github_issue_number + '';
  var caseId       = inputs.case_sys_id + '';
  var crNumber     = inputs.cr_number + '';
  var crSysId      = inputs.cr_sys_id + '';
  var assignedTo   = inputs.cr_assigned_to + '';
  var plannedStart = inputs.cr_planned_start + '';
  var plannedEnd   = inputs.cr_planned_end + '';

  var dataPillState = inputs.cr_state + '';
  var crState       = dataPillState;
  var crGr = new GlideRecord('change_request');
  if (crGr.get(crSysId)) {
    crState = crGr.getValue('state') + '';
  }

  var action;
  if (dataPillState !== crState) {
    action = (crState === '-3' || crState === '-2') ? 'approval_updated' : 'state_changed';
  } else {
    action = 'dates_updated';
  }

  outputs.issue_number   = issueNumber;
  outputs.case_sys_id    = caseId;
  outputs.cr_number      = crNumber;
  outputs.cr_sys_id      = crSysId;
  outputs.cr_state       = crState;
  outputs.assigned_to    = assignedTo;
  outputs.planned_start  = plannedStart;
  outputs.planned_end    = plannedEnd;
  outputs.action         = action;
  outputs.should_send    = (issueNumber.length > 0 && crNumber.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

---

### 2.2 Add If Condition — Approval States

Click **+** below Script Step 1. Select **Flow Logic > If**.

- Data pill: Script Step 1 > `action`
- Operator: `is`
- Value: `approval_updated`

Click **Done**. The next step goes inside the **then** branch.

---

### 2.3 Call the Subflow inside the then branch

Click **+** inside the **then** branch. Select **Subflow**.

Search for and select **SN CR Approval Lookup**.

#### Input Variables

| Variable Name | Source |
|---|---|
| `cr_sys_id` | Script Step 1 > `cr_sys_id` |
| `cr_state` | Script Step 1 > `cr_state` |

Click **Done**.

---

### 2.4 Update Script Step 2

Open the existing **Script Step 2** (inside the `should_send is true` If branch).

#### Add two new input variables

| Variable Name | Type | Source |
|---|---|---|
| `authorized_by` | String | If (approval_updated) > Then > SN CR Approval Lookup > `authorized_by` |
| `cab_approvers` | String | If (approval_updated) > Then > SN CR Approval Lookup > `cab_approvers` |

> When the action is not `approval_updated` these data pills resolve to empty strings — that is expected and safe.

#### Update the request body

Replace the `rm.setRequestBody(...)` call with:

```javascript
rm.setRequestBody(JSON.stringify({
  event_type: 'servicenow-cr-update',
  client_payload: {
    github_issue_number: inputs.issue_number,
    cr_number:           inputs.cr_number,
    cr_sys_id:           inputs.cr_sys_id,
    cr_state:            inputs.cr_state,
    assigned_to:         inputs.assigned_to,
    case_sys_id:         inputs.case_sys_id,
    planned_start:       inputs.planned_start,
    planned_end:         inputs.planned_end,
    action:              inputs.action,
    authorized_by:       inputs.authorized_by || '',
    cab_approvers:       inputs.cab_approvers || ''
  }
}));
```

Click **Done**.

---

### 2.5 Save and Activate Flow B

Click **Save**, then **Activate**.

---

## Part 3 — GitHub Actions: sn-cr-notifier.yml

The `sn-cr-notifier.yml` workflow needs two changes — both inside the existing `script` block of the `cr-update` job:

1. Accept `approval_updated` as a valid action
2. Post a formatted comment showing the authorizer or CAB approvers

See the diff applied to `.github/workflows/sn-cr-notifier.yml` in this same commit.

---

## Testing

### Test — Authorize state

1. Open a Change Request linked to a case with a value in `u_github_issue_number`
2. Change **State** to **Authorize** and save
3. Go to the GitHub issue — within ~30 seconds a comment appears:

```
💬 ServiceNow Change Request — Authorized #CHG0012345 [timestamp]

> State: Authorize
> Authorized by: John Smith

---
*Automatically notified by ServiceNow*
```

### Test — CAB Approval state

1. Move the same CR to **CAB Approval** and ensure CAB approval records exist
2. Go to the GitHub issue:

```
💬 ServiceNow Change Request — CAB Approved #CHG0012345 [timestamp]

> State: CAB Approval
> CAB Approved by: Jane Doe, Bob Lee

---
*Automatically notified by ServiceNow*
```

### Test — Other state changes still work

1. Move a CR to **Implement** (state `0`)
2. The comment should still show `State: Implement` — the existing `state_changed` path fires, not `approval_updated`

---

## Checking execution logs

1. Go to: `All > Process Automation > Flow Designer`
2. Click the **Executions** tab
3. Find **SN CR Updated → GitHub** and click the execution row
4. Expand **If (approval_updated) > SN CR Approval Lookup** to see the subflow's `authorized_by` and `cab_approvers` outputs

---

## Troubleshooting

| Problem | What to check |
|---|---|
| No comment on Authorize / CAB Approval | Confirm Script Step 1 in Flow B sets `action = 'approval_updated'` for states `-3` and `-2` |
| `authorized_by` is empty | The `sysapproval_approver` record may not be committed yet — check `u_authorized_by` / `approved_by` directly on the CR |
| `cab_approvers` is empty | No `sysapproval_approver` records with `state = approved` exist yet — confirm CAB approvals were recorded |
| Other state changes now show `approval_updated` | Confirm the `crState === '-3' \|\| crState === '-2'` check uses the live `crState` value, not `dataPillState` |
| Subflow step not visible in Flow B | The subflow must be **activated** before editing Flow B |
| `approval_updated` logged as unknown action | Confirm `sn-cr-notifier.yml` is updated and on the default branch |
