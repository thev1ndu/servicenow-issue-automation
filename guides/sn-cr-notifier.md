# Guide: ServiceNow CR Notifier → GitHub (Flow Designer)

When a Change Request is created from a ServiceNow case, or when its state changes,
this flow posts a notification on the linked GitHub issue.

GitHub Actions file that receives it: `.github/workflows/sn-cr-notifier.yml`
GitHub event types sent:
- `servicenow-cr-update` with `action: created`
- `servicenow-cr-update` with `action: state_changed`

---

## What happens end to end

**On CR created:**
1. Agent clicks "Create Project Change Request" on a case
2. ServiceNow creates a `change_request` record linked to the case via the `Parent` field
3. Flow A fires, looks up the parent case, finds the GitHub issue number
4. Calls GitHub — `sn-cr-notifier.yml` posts a CR created notification on the issue

**On CR state change:**
1. Agent moves the CR from one state to another (e.g. Assess → Implement)
2. Flow B fires, same lookup and GitHub call with `action: state_changed`

---

## Before you start

Complete [setup-rest-message.md](setup-rest-message.md) first.
That guide creates the `GitHub Integration` REST Message with the `dispatch_cr` HTTP Method that both flows use.

Also confirm:
- The case table has a field `u_github_issue_number` (String)
- You have Flow Designer author role

---

## Flow A — CR Created

### A.1 Create the Flow

Go to: `All > Process Automation > Flow Designer`

Click **New > Flow**

- **Name**: `SN CR Created → GitHub`
- **Description**: `Notifies GitHub when a Change Request is created from a case`
- **Run as**: `System User`

Click **Submit**.

---

### A.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Created**
2. **Table**: `Change Request [change_request]`
3. Under **Condition**, click **Add Filters**:
   - Field: `Parent` | Operator: `is not empty`

   This ensures the flow only fires for CRs that are linked to a case. Standalone CRs with no parent are ignored.

Click **Done**.

---

### A.3 Look up the parent Case record

The Change Request has a `Parent` field that points to the Customer Service Case.
You need to look it up to read `u_github_issue_number` from the case.

Click **+** below the trigger.

Search for and select **Look Up Record** (under ServiceNow Core).

Configure it:

| Field | Value |
|---|---|
| **Table** | `Customer Service Case [sn_customerservice_case]` |
| **Filter field** | `Sys ID` |
| **Operator** | `is` |
| **Value** | Click the data pill icon → expand **Trigger > Change Request Record > Parent** → select **Sys ID** |

Click **Done**.

This step will output the full Case record. In later steps it appears in the data picker as **Look Up Record > Customer Service Case Record**.

---

### A.4 Add Script Step 1 — Read values and check if linked

Click **+** below the Look Up Record step. Select **Script**.

#### Input Variables

| Variable Name | Type | Data pill to select |
|---|---|---|
| `github_issue_number` | String | Look Up Record > Customer Service Case Record > **u_github_issue_number** |
| `case_sys_id` | String | Look Up Record > Customer Service Case Record > **Sys ID** |
| `cr_number` | String | Trigger > Change Request Record > **Number** |
| `cr_sys_id` | String | Trigger > Change Request Record > **Sys ID** |
| `cr_state` | String | Trigger > Change Request Record > **State** |
| `cr_environment` | String | Trigger > Change Request Record > **Environment** *(skip if the field does not exist on your CR table)* |

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseId      = inputs.case_sys_id + '';
  var crNumber    = inputs.cr_number + '';
  var crSysId     = inputs.cr_sys_id + '';
  var crState     = inputs.cr_state + '';
  var crEnv       = (inputs.cr_environment + '') || 'Not specified';

  outputs.issue_number   = issueNumber;
  outputs.case_sys_id    = caseId;
  outputs.cr_number      = crNumber;
  outputs.cr_sys_id      = crSysId;
  outputs.cr_state       = crState;
  outputs.cr_environment = crEnv;

  // Only send if the case has a GitHub issue number AND we have a CR number
  outputs.should_send = (issueNumber.length > 0 && crNumber.length > 0) ? 'true' : 'false';

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
| `cr_environment` | String |
| `should_send` | String |

Click **Done**.

---

### A.5 Add an If Condition

Click **+** below Script Step 1. Select **Flow Logic > If**

- Data pill: Script Step 1 > `should_send`
- Operator: `is`
- Value: `true`

Click **Done**. Place the next step inside the **then** branch.

---

### A.6 Add Script Step 2 — Call GitHub inside the then branch

Click **+** inside the **then** branch. Select **Script**.

#### Input Variables

| Variable Name | Type | Data pill to select |
|---|---|---|
| `issue_number` | String | Script Step 1 > `issue_number` |
| `case_sys_id` | String | Script Step 1 > `case_sys_id` |
| `cr_number` | String | Script Step 1 > `cr_number` |
| `cr_sys_id` | String | Script Step 1 > `cr_sys_id` |
| `cr_state` | String | Script Step 1 > `cr_state` |
| `cr_environment` | String | Script Step 1 > `cr_environment` |

#### Script

```javascript
(function execute(inputs, outputs) {

  try {
    // 'GitHub Integration' is the REST Message name
    // 'dispatch_cr' is the HTTP Method name
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch_cr');

    rm.setStringParameterNoEscape('issue_number',   inputs.issue_number);
    rm.setStringParameterNoEscape('cr_number',      inputs.cr_number);
    rm.setStringParameterNoEscape('cr_sys_id',      inputs.cr_sys_id);
    rm.setStringParameterNoEscape('cr_state',       inputs.cr_state);
    rm.setStringParameterNoEscape('previous_state', '');
    rm.setStringParameterNoEscape('cr_environment', inputs.cr_environment);
    rm.setStringParameterNoEscape('case_sys_id',    inputs.case_sys_id);
    rm.setStringParameterNoEscape('action',         'created');

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.http_status = httpStatus + '';
    outputs.success     = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub CR dispatch failed. HTTP ' + httpStatus + ' Body: ' + response.getBody());
    }

  } catch (ex) {
    outputs.http_status = 'error';
    outputs.success     = 'false';
    gs.error('GitHub CR dispatch exception: ' + ex.message);
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

### A.7 Save and Activate Flow A

Click **Save**, then **Activate**.

---

## Flow B — CR State Changed

This is a separate flow. The structure is identical to Flow A except the trigger and the `previous_state` field.

### B.1 Create the Flow

Click **New > Flow**

- **Name**: `SN CR State Changed → GitHub`
- **Description**: `Notifies GitHub when a Change Request state changes`
- **Run as**: `System User`

---

### B.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Updated**
2. **Table**: `Change Request [change_request]`
3. **Condition**:
   - Field: `State` | Operator: `changes`
   - AND Field: `Parent` | Operator: `is not empty`

Click **Done**.

---

### B.3 Look up the parent Case record

Exactly the same as Step A.3:

- **Look Up Record** step
- Table: `Customer Service Case [sn_customerservice_case]`
- Filter: `Sys ID` is → `Trigger > Change Request Record > Parent > Sys ID`

---

### B.4 Add Script Step 1 — Read values

Same input variables as A.4, with one additional variable for previous state:

| Variable Name | Type | Data pill to select |
|---|---|---|
| `github_issue_number` | String | Look Up Record > Customer Service Case Record > **u_github_issue_number** |
| `case_sys_id` | String | Look Up Record > Customer Service Case Record > **Sys ID** |
| `cr_number` | String | Trigger > Change Request Record > **Number** |
| `cr_sys_id` | String | Trigger > Change Request Record > **Sys ID** |
| `cr_state` | String | Trigger > Change Request Record > **State** |
| `cr_previous_state` | String | Trigger > Change Request Record > **State (Previous value)** |

> "**State (Previous value)**" is available in the data picker for Record Updated triggers.
> In the data pill panel, expand **Trigger > Change Request Record**, then look for the field with "(Previous value)" suffix.

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber   = inputs.github_issue_number + '';
  var caseId        = inputs.case_sys_id + '';
  var crNumber      = inputs.cr_number + '';
  var crSysId       = inputs.cr_sys_id + '';
  var crState       = inputs.cr_state + '';
  var previousState = inputs.cr_previous_state + '';

  outputs.issue_number    = issueNumber;
  outputs.case_sys_id     = caseId;
  outputs.cr_number       = crNumber;
  outputs.cr_sys_id       = crSysId;
  outputs.cr_state        = crState;
  outputs.previous_state  = previousState;
  outputs.should_send     = (issueNumber.length > 0 && crNumber.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

#### Output Variables

Same as A.4 outputs, replacing `cr_environment` with `previous_state`:

| Variable Name | Type |
|---|---|
| `issue_number` | String |
| `case_sys_id` | String |
| `cr_number` | String |
| `cr_sys_id` | String |
| `cr_state` | String |
| `previous_state` | String |
| `should_send` | String |

---

### B.5 Add an If Condition

Same as A.5 — check `should_send` is `true`.

---

### B.6 Add Script Step 2 — Call GitHub

Same structure as A.6 with this script:

```javascript
(function execute(inputs, outputs) {

  try {
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch_cr');

    rm.setStringParameterNoEscape('issue_number',   inputs.issue_number);
    rm.setStringParameterNoEscape('cr_number',      inputs.cr_number);
    rm.setStringParameterNoEscape('cr_sys_id',      inputs.cr_sys_id);
    rm.setStringParameterNoEscape('cr_state',       inputs.cr_state);
    rm.setStringParameterNoEscape('previous_state', inputs.previous_state);
    rm.setStringParameterNoEscape('cr_environment', '');
    rm.setStringParameterNoEscape('case_sys_id',    inputs.case_sys_id);
    rm.setStringParameterNoEscape('action',         'state_changed');

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.http_status = httpStatus + '';
    outputs.success     = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub CR state dispatch failed. HTTP ' + httpStatus + ' Body: ' + response.getBody());
    }

  } catch (ex) {
    outputs.http_status = 'error';
    outputs.success     = 'false';
    gs.error('GitHub CR state dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

---

### B.7 Save and Activate Flow B

Click **Save**, then **Activate**.

---

## Testing

### Test Flow A

1. Open a Customer Service Case with a value in `u_github_issue_number`
2. Click **Create Project Change Request** in the toolbar
3. Fill in required fields and save the CR
4. Go to the GitHub issue — within ~30 seconds:

```
New Change Request Created

CR Number: CHG0012345
State: Assess
Environment: Production

All Change Requests for this Case (1)
- CHG0012345 - Assess

---
*Automatically notified by ServiceNow*
```

### Test Flow B

1. Open an existing Change Request linked to a case with a GitHub issue
2. Change the **State** field and save
3. Go to the GitHub issue:

```
Change Request State Updated

CR Number: CHG0012345
State Change: Assess → Implement

---
*Automatically notified by ServiceNow*
```

---

## Checking execution logs

1. Go to: `All > Process Automation > Flow Designer`
2. Click the **Executions** tab
3. Find your flow name and click the execution row
4. Expand Script Step 2 — the `http_status` output shows what GitHub returned

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow A does not fire | Confirm the trigger is **Record Created** on `change_request`, not Record Updated |
| Flow B does not fire | Confirm the trigger is **Record Updated** with condition `State changes` |
| Look Up Record returns no record | The CR's `Parent` field is not set — the CR must be created from within the case using the toolbar button, not created independently |
| `should_send` is `false` | The parent case has no value in `u_github_issue_number` |
| `http_status` is `401` | The Github Basic Auth profile has a wrong or expired PAT — update it |
| `http_status` is `404` | The org/repo in the REST Message Endpoint URL is wrong |
| `http_status` is `422` | The `sn-cr-notifier.yml` workflow file is not on the default branch |
| `previous_state` is empty | In the data picker, make sure you selected "State **(Previous value)**" not just "State" |
