# Guide: ServiceNow Case Updates → GitHub (Flow Designer)

A single Flow Designer flow that covers two case events: when a case is **closed**, and
when a case is **assigned** to a team member. Both events post a comment on the linked
GitHub issue. The close event also closes the GitHub issue automatically.

GitHub Actions file that receives it: `.github/workflows/sn-case-updates.yml`
GitHub event type sent: `servicenow-case-update`
Actions handled: `closed`, `assigned`

---

## What happens end to end

**On case closed:**
1. Agent changes the Case **State** to `Closed` and saves
2. The flow fires, reads the case fields, sets `action = "closed"`
3. A `servicenow-case-update` dispatch is sent to GitHub
4. `sn-case-updates.yml` posts a `✅ ServiceNow Case Closed` comment and closes the GitHub issue

**On case assigned:**
1. Agent sets or changes the Case **Assigned to** field and saves
2. The flow fires, reads the case fields, sets `action = "assigned"`
3. A `servicenow-case-update` dispatch is sent to GitHub
4. `sn-case-updates.yml` posts a `👤 ServiceNow Case Assigned` comment on the issue

---

## Before you start

Complete [setup-rest-message.md](setup-rest-message.md) first.
That guide creates the `GitHub Integration` REST Message with the `dispatch` HTTP Method that this flow uses.

Also confirm:
- The case table has a field `u_github_issue_number` (String) that stores the GitHub issue number
- You have Flow Designer author role

---

## Part 1 — Create the Flow

### 1.1 Open Flow Designer

Go to: `All > Process Automation > Flow Designer`

Click **New > Flow**

Fill in:
- **Name**: `SN Case Updates → GitHub`
- **Description**: `Notifies GitHub when a case is closed or assigned`
- **Run as**: `System User`

Click **Submit**. The flow editor opens.

---

### 1.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Updated**
2. **Table**: type and select `Customer Service Case [sn_customerservice_case]`
3. Under **Condition**, click **Add Filters** and set up the following rows:

   | Row | Field | Operator | Value | Connector |
   |-----|-------|----------|-------|-----------|
   | 1 | `u_github_issue_number` | `is not empty` | — | AND |
   | 2 | `State` | `changes to` | `Closed` | OR |
   | 3 | `Assigned to` | `changes` | — | — |

   To add the OR connector: after adding row 2, click the **AND** connector between rows 2 and 3 and switch it to **OR**.

4. Leave **Run trigger** as: `For every update that causes condition to be true`

Click **Done**.

> **Finding the Closed state value:** Open any closed case, right-click the **State** label, choose **Show value**. Use that display label in the `changes to` filter — Flow Designer resolves it to the correct integer. Common values are `3` or `5` depending on the instance configuration.

---

### 1.3 Add Script Step 1 — Determine what changed

Click **+** below the trigger. Select **Script**.

#### Input Variables

| Variable Name | Type | Data pill to select |
|---|---|---|
| `github_issue_number` | String | Trigger > Customer Service Case Record > **u_github_issue_number** |
| `case_number` | String | Trigger > Customer Service Case Record > **Number** |
| `case_sys_id` | String | Trigger > Customer Service Case Record > **Sys ID** |
| `case_resolution` | String | Trigger > Customer Service Case Record > **Resolution notes** |
| `assigned_to_name` | String | Trigger > Customer Service Case Record > **Assigned to** > **Name** |
| `account_name` | String | Trigger > Customer Service Case Record > **Account** > **Name** |

> If your instance does not have a **Resolution notes** field, omit `case_resolution` from the input variables and remove the `outputs.resolution_notes` line in the script below.

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseNumber  = inputs.case_number + '';
  var caseSysId   = inputs.case_sys_id + '';
  var resolution  = inputs.case_resolution + '';
  var assignedTo  = inputs.assigned_to_name + '';
  var accountName = inputs.account_name + '';

  // Read the current state display value directly from the record
  // to avoid hardcoding integer state codes across different instances.
  var gr = new GlideRecord('sn_customerservice_case');
  gr.get(caseSysId);
  var stateDisplay = gr.state.getDisplayValue() + '';
  var isClosed = (stateDisplay.toLowerCase() === 'closed');

  // Check if Assigned to changed in this update via the audit log.
  // The trigger may fire for either condition, so we detect which one applies.
  var isAssigned = false;
  if (!isClosed && assignedTo.length > 0) {
    var audit = new GlideRecord('sys_audit');
    audit.addQuery('tablename', 'sn_customerservice_case');
    audit.addQuery('documentkey', caseSysId);
    audit.addQuery('fieldname', 'assigned_to');
    audit.orderByDesc('sys_created_on');
    audit.setLimit(1);
    audit.query();
    if (audit.next()) {
      isAssigned = true;
    }
  }

  // Closed takes priority if both somehow happen in the same save.
  var action = isClosed ? 'closed' : (isAssigned ? 'assigned' : '');

  outputs.issue_number     = issueNumber;
  outputs.case_number      = caseNumber;
  outputs.case_sys_id      = caseSysId;
  outputs.resolution_notes = resolution;
  outputs.assigned_to      = assignedTo;
  outputs.sn_user          = gs.getUserDisplayName();
  outputs.action           = action;
  outputs.account_name     = accountName;
  outputs.should_send      = (issueNumber.length > 0 && action.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

#### Output Variables

| Variable Name | Type |
|---|---|
| `issue_number` | String |
| `case_number` | String |
| `case_sys_id` | String |
| `resolution_notes` | String |
| `assigned_to` | String |
| `sn_user` | String |
| `action` | String |
| `account_name` | String |
| `should_send` | String |

Click **Done**.

---

### 1.4 Add an If Condition — skip if nothing to send

Click **+** below Script Step 1.

Select **Flow Logic > If**

Configure:
- Data pill: **Script step 1 > `should_send`**
- Operator: `is`
- Value: `true`

Click **Done**.

All remaining steps go inside the **then** branch.

---

### 1.5 Add Script Step 2 — Call the GitHub REST Message

Click **+** inside the **then** branch. Select **Script**.

#### Input Variables

| Variable Name | Type | Data pill to select |
|---|---|---|
| `issue_number` | String | Script step 1 > `issue_number` |
| `case_number` | String | Script step 1 > `case_number` |
| `case_sys_id` | String | Script step 1 > `case_sys_id` |
| `resolution_notes` | String | Script step 1 > `resolution_notes` |
| `assigned_to` | String | Script step 1 > `assigned_to` |
| `sn_user` | String | Script step 1 > `sn_user` |
| `action` | String | Script step 1 > `action` |
| `account_name` | String | Script step 1 > `account_name` |

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

    var config = JSON.parse(configJson)[inputs.account_name];
    if (!config) {
      gs.error('No config entry for account "' + inputs.account_name + '" in github.dispatch.config');
      outputs.http_status = 'config_missing';
      outputs.success     = 'false';
      return;
    }
    var endpoint = 'https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches';

    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
    rm.setEndpoint(endpoint);
    rm.setRequestHeader('Authorization', 'token ' + config.token);

    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-case-update',
      client_payload: {
        action:              inputs.action,
        github_issue_number: inputs.issue_number,
        case_number:         inputs.case_number,
        case_sys_id:         inputs.case_sys_id,
        resolution_notes:    inputs.resolution_notes,
        assigned_to:         inputs.assigned_to,
        sn_user:             inputs.sn_user
      }
    }));

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.http_status = httpStatus + '';
    outputs.success     = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub case-update dispatch failed. HTTP ' + httpStatus + ' Body: ' + response.getBody());
    }

  } catch (ex) {
    outputs.http_status = 'error';
    outputs.success     = 'false';
    gs.error('GitHub case-update dispatch exception: ' + ex.message);
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

### 1.6 Save and Activate

1. Click **Save** (top right)
2. Click **Activate**

The flow is now live and handles both events.

---

## Part 2 — What GitHub does when it receives the event

`sn-case-updates.yml` reads the `action` field and branches accordingly.

### action = `"closed"`

1. Posts this comment on the issue:

```
✅ ServiceNow Case Closed — [CS0001234](link)

> Case has been closed in ServiceNow.
> Resolution: <resolution_notes if provided>
> 🕐 Closed: Wed, 14 May 2026 10:30:00 IST

---
*Closed by Jane Smith via ServiceNow*
```

2. Calls `issues.update` with `state: closed, state_reason: completed` — the GitHub issue is closed immediately.

### action = `"assigned"`

Posts this comment on the issue (issue stays open):

```
👤 ServiceNow Case Assigned — [CS0001234](link)

> Assigned to: John Smith
> 🕐 Updated: Wed, 14 May 2026 09:15:00 IST

---
*Assigned by Jane Smith via ServiceNow*
```

---

## Testing

### Manual test from Script Background

Test both actions without modifying a real case.

Go to: `All > System Definition > Scripts - Background`

**Test the assigned action** (replace `1` with a real open GitHub issue number):

```javascript
var configJson = gs.getProperty('github.dispatch.config');
var REPO   = 'servicenow-issue-automation';
var config = JSON.parse(configJson)[REPO];

var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + REPO + '/dispatches');
rm.setRequestHeader('Authorization', 'token ' + config.token);
rm.setRequestBody(JSON.stringify({
  event_type: 'servicenow-case-update',
  client_payload: {
    action:              'assigned',
    github_issue_number: '1',
    case_number:         'CS0000001',
    case_sys_id:         'abc123def456abc123def456abc123de',
    assigned_to:         'John Smith',
    sn_user:             gs.getUserDisplayName()
  }
}));

var response = rm.execute();
gs.info('HTTP Status: ' + response.getStatusCode());
```

Expected: `HTTP Status: 204`. A `👤 ServiceNow Case Assigned` comment should appear on GitHub issue #1 within ~30 seconds.

**Test the closed action** (use a different issue number you're OK closing):

```javascript
var configJson = gs.getProperty('github.dispatch.config');
var REPO   = 'servicenow-issue-automation';
var config = JSON.parse(configJson)[REPO];

var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + REPO + '/dispatches');
rm.setRequestHeader('Authorization', 'token ' + config.token);
rm.setRequestBody(JSON.stringify({
  event_type: 'servicenow-case-update',
  client_payload: {
    action:              'closed',
    github_issue_number: '2',
    case_number:         'CS0000001',
    case_sys_id:         'abc123def456abc123def456abc123de',
    resolution_notes:    'Issue resolved after investigation.',
    sn_user:             gs.getUserDisplayName()
  }
}));

var response = rm.execute();
gs.info('HTTP Status: ' + response.getStatusCode());
```

Expected: `HTTP Status: 204`. GitHub issue #2 should be closed with a `✅ ServiceNow Case Closed` comment.

### Live test from a case

**For assigned:**
1. Open a case with `u_github_issue_number` filled
2. Set the **Assigned to** field to any agent and save
3. Go to the GitHub issue — assignment comment appears within ~30 seconds

**For closed:**
1. Open a case with `u_github_issue_number` filled
2. Change **State** to `Closed` and save
3. Go to the GitHub issue — closure comment appears and the issue is closed within ~30 seconds

---

## Checking execution logs

1. Go to: `All > Process Automation > Flow Designer`
2. Click the **Executions** tab
3. Find `SN Case Updates → GitHub` and click the execution row
4. Expand **Script step 1** — check `action` and `should_send` outputs
5. Expand **Script step 2** — `http_status` shows what GitHub returned

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow does not trigger on close | Confirm the trigger condition includes `State changes to Closed` on `sn_customerservice_case` — verify the state display label matches your instance |
| Flow does not trigger on assignment | Confirm the trigger condition includes `Assigned to changes` with an OR connector, not AND |
| `action` is empty in Script step 1 | `isClosed` was false and the audit log found no `assigned_to` change — confirm audit is enabled for `sn_customerservice_case.assigned_to` |
| `should_send` is `false` | The case has no value in `u_github_issue_number` |
| Only close fires when both happen | Expected — `closed` takes priority over `assigned` in the same save |
| `http_status` is `401` | Token in `github.dispatch.config` is expired or missing `repo` scope |
| `http_status` is `404` | Owner or repo in `github.dispatch.config` is wrong, or the issue number does not exist |
| `http_status` is `422` | The `sn-case-updates.yml` workflow is not on the default branch |
| Flow fires on Resolved, not Closed | Your instance uses a different state label — check the display value and update the trigger condition |
| Flow fires twice | A second automation is also changing the same field. Add a trigger condition: `Updated by` is `User` to exclude programmatic updates |
