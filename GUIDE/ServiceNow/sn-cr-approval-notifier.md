# Guide: CR Approval Updated → GitHub

When the **Authorized by** field is set on a Change Request, or its **state moves to Scheduled
(CAB Approved)**, this flow posts a notification on the linked GitHub issue.

GitHub Actions file that receives it: `.github/workflows/sn-cr-notifier.yml`
GitHub event type sent: `servicenow-cr-update` with `action: approval_updated`

---

## What happens end to end

**When `u_authorized_by` is set (Authorize path):**
1. Agent fills the **Authorized by** field and saves the CR
2. The flow fires, reads the current `u_authorized_by` display value
3. Dispatches to GitHub with `approval_type: authorized`, `authorized_by: <name>`
4. `sn-cr-notifier.yml` posts a 🔐 comment on the issue

**When state moves to `-2` Scheduled (CAB Approved path):**
1. The CR state is saved as `-2`
2. The same flow fires, detects state `-2`, queries `sysapproval_approver` for all approved approvers
3. Dispatches with `approval_type: cab_approved`, `cab_approvers: <names>`
4. `sn-cr-notifier.yml` posts a ✅ comment on the issue

If both conditions are true in the same save (state hits `-2` and `u_authorized_by` is also set),
the **CAB Approved** path takes priority.

---

## Prerequisites

- [setup-rest-message.md](setup-rest-message.md) complete — `GitHub Integration` REST Message with `dispatch_cr` method must exist
- [sn-cr-notifier.md](sn-cr-notifier.md) complete — `sn-cr-notifier.yml` workflow must be on the default branch
- `sn-cr-notifier.yml` updated to handle `approval_updated` action (see commit that introduced this guide)

---

## Flow C — SN CR Approval Updated → GitHub

### C.1 Create the Flow

Go to: `All > Process Automation > Flow Designer`

Click **New > Flow**

- **Name**: `SN CR Approval Updated → GitHub`
- **Description**: `Notifies GitHub when Authorized by is set or state moves to CAB Approved`
- **Run as**: `System User`

Click **Submit**.

---

### C.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Updated**
2. **Table**: `Change Request [change_request]`
3. Under **Condition**, click **Add Filters** and build the following (use **OR** between the first two rows):

```
u_authorized_by  changes
OR
State            changes to    -2 (Scheduled)
AND
Parent           is not empty
```

> In Flow Designer, click **or** at the left of each row to switch between AND/OR grouping.
> The `Parent is not empty` line must remain as AND — it applies to both paths.

Click **Done**.

---

### C.3 Look up the parent Case record

Click **+** below the trigger. Select **Look Up Record**.

| Field | Value |
|---|---|
| **Table** | `Customer Service Case [sn_customerservice_case]` |
| **Filter field** | `Sys ID` |
| **Operator** | `is` |
| **Value** | Trigger > Change Request Record > **Parent** > **Sys ID** |

Click **Done**.

---

### C.4 Add Script Step 1 — Determine approval type and collect names

Click **+** below Look Up Record. Select **Script**.

#### Input Variables

| Variable Name | Type | Data pill to select |
|---|---|---|
| `github_issue_number` | String | Look Up Record > Customer Service Case Record > **u_github_issue_number** |
| `case_sys_id` | String | Look Up Record > Customer Service Case Record > **Sys ID** |
| `cr_number` | String | Trigger > Change Request Record > **Number** |
| `cr_sys_id` | String | Trigger > Change Request Record > **Sys ID** |

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseId      = inputs.case_sys_id + '';
  var crNumber    = inputs.cr_number + '';
  var crSysId     = inputs.cr_sys_id + '';

  var approvalType = '';
  var authorizedBy = '';
  var cabApprovers = '';
  var crState      = '';

  // Read live CR values
  var crGr = new GlideRecord('change_request');
  if (crGr.get(crSysId)) {
    crState = crGr.getValue('state') + '';

    if (crState === '-2') {
      // CAB Approved path — collect all approved CAB approvers
      approvalType = 'cab_approved';
      var appGr = new GlideRecord('sysapproval_approver');
      appGr.addQuery('source_table', 'change_request');
      appGr.addQuery('document_id', crSysId);
      appGr.addQuery('state', 'approved');
      appGr.orderBy('sys_updated_on');
      appGr.query();
      var names = [];
      while (appGr.next()) {
        var n = appGr.getDisplayValue('approver');
        if (n) names.push(n);
      }
      cabApprovers = names.join(', ');
    } else {
      // Authorized by path
      authorizedBy = crGr.getDisplayValue('u_authorized_by');
      if (authorizedBy) {
        approvalType = 'authorized';
      }
    }
  }

  outputs.issue_number   = issueNumber;
  outputs.case_sys_id    = caseId;
  outputs.cr_number      = crNumber;
  outputs.cr_sys_id      = crSysId;
  outputs.cr_state       = crState;
  outputs.approval_type  = approvalType;
  outputs.authorized_by  = authorizedBy;
  outputs.cab_approvers  = cabApprovers;
  outputs.should_send    = (issueNumber.length > 0 && crNumber.length > 0 && approvalType.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

#### Output Variables

| Variable Name | Type |
|---|---|
| `issue_number` | String |
| `case_sys_id` | String |
| `cr_number` | String |
| `cr_sys_id` | String |
| `cr_state` | String |
| `approval_type` | String |
| `authorized_by` | String |
| `cab_approvers` | String |
| `should_send` | String |

Click **Done**.

---

### C.5 Add an If Condition

Click **+** below Script Step 1. Select **Flow Logic > If**.

- Data pill: Script Step 1 > `should_send`
- Operator: `is`
- Value: `true`

Click **Done**. Place the next step inside the **then** branch.

---

### C.6 Add Script Step 2 — Call GitHub

Click **+** inside the **then** branch. Select **Script**.

#### Input Variables

| Variable Name | Type | Source |
|---|---|---|
| `issue_number` | String | Script Step 1 > `issue_number` |
| `case_sys_id` | String | Script Step 1 > `case_sys_id` |
| `cr_number` | String | Script Step 1 > `cr_number` |
| `cr_sys_id` | String | Script Step 1 > `cr_sys_id` |
| `cr_state` | String | Script Step 1 > `cr_state` |
| `approval_type` | String | Script Step 1 > `approval_type` |
| `authorized_by` | String | Script Step 1 > `authorized_by` |
| `cab_approvers` | String | Script Step 1 > `cab_approvers` |

#### Script

```javascript
(function execute(inputs, outputs) {

  try {
    var configJson = gs.getProperty('github.dispatch.config');
    if (!configJson) {
      gs.error('System property github.dispatch.config is missing');
      outputs.http_status = 'config_missing';
      outputs.success     = 'false';
      return;
    }

    var config   = JSON.parse(configJson);
    var endpoint = 'https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches';

    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch_cr');
    rm.setEndpoint(endpoint);
    rm.setRequestHeader('Authorization', 'token ' + config.token);

    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-cr-update',
      client_payload: {
        github_issue_number: inputs.issue_number,
        cr_number:           inputs.cr_number,
        cr_sys_id:           inputs.cr_sys_id,
        cr_state:            inputs.cr_state,
        case_sys_id:         inputs.case_sys_id,
        action:              'approval_updated',
        approval_type:       inputs.approval_type,
        authorized_by:       inputs.authorized_by || '',
        cab_approvers:       inputs.cab_approvers || ''
      }
    }));

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.http_status = httpStatus + '';
    outputs.success     = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub approval dispatch failed. HTTP ' + httpStatus + ' Body: ' + response.getBody());
    }

  } catch (ex) {
    outputs.http_status = 'error';
    outputs.success     = 'false';
    gs.error('GitHub approval dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

#### Output Variables

| Variable Name | Type |
|---|---|
| `http_status` | String |
| `success` | String |

Click **Done**.

---

### C.7 Save and Activate

Click **Save**, then **Activate**.

---

## GitHub comment examples

### Authorized by set

```
🔐 ServiceNow Change Request — Authorized #CHG0012345 [2026-05-15 14:32:00 IST]

> Authorized by: John Smith

---
*Automatically notified by ServiceNow*
```

### State moved to Scheduled (CAB Approved)

```
✅ ServiceNow Change Request — CAB Approved #CHG0012345 [2026-05-15 15:10:00 IST]

> CAB Approved by: Jane Doe, Bob Lee

---
*Automatically notified by ServiceNow*
```

---

## Testing

### Test — Authorized by path

1. Open a Change Request linked to a case with a value in `u_github_issue_number`
2. Set the **Authorized by** (`u_authorized_by`) field to any user and save (state should NOT be `-2`)
3. Go to the GitHub issue — within ~30 seconds the 🔐 comment appears

### Test — CAB Approved path

1. Open the same CR and change state to **Scheduled** (`-2`), ensuring CAB approval records exist
2. Save — the flow fires, collects all approved `sysapproval_approver` records, and dispatches
3. Go to the GitHub issue — the ✅ comment appears listing all approved CAB approvers

### Test — Both conditions in one save

1. Set `u_authorized_by` and change state to `-2` in the same save
2. The CAB Approved path takes priority — only a ✅ comment is posted (`should_send` depends on `approvalType`)

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow does not fire on `u_authorized_by` change | Confirm the OR trigger condition includes `u_authorized_by changes` (not just state) |
| Flow does not fire on state `-2` | Confirm `State changes to -2` is in the trigger condition; check the OR grouping |
| `approval_type` is empty | Neither `u_authorized_by` nor state `-2` resolved — check `crGr.get(crSysId)` returned true in the execution log |
| `authorized_by` is empty | `u_authorized_by` on the live CR record is not set; confirm the field was saved before the flow ran |
| `cab_approvers` is empty | No `sysapproval_approver` records with `state = approved` for this CR; confirm CAB approvals are recorded |
| Look Up Record returns empty | The CR's `Parent` field is not set — the CR must be created from within a case |
| `should_send` is `false` | The parent case has no value in `u_github_issue_number` |
| `http_status` is `422` | `sn-cr-notifier.yml` is not on the default branch or the `approval_updated` handler is missing |
