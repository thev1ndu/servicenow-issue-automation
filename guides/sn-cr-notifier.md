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
1. An agent clicks "Create Project Change Request" on a case
2. ServiceNow creates a `change_request` record with `parent` pointing to the case
3. Flow A fires because a Change Request record was created
4. It looks up the parent case to find the GitHub issue number
5. Calls GitHub API → `sn-cr-notifier.yml` posts a notification on the GitHub issue

**On CR state change:**
1. An agent moves a CR from one state to another (e.g., Assess → Implement)
2. Flow B fires because the `state` field changed on the Change Request
3. Same lookup and GitHub call as above, with `action: state_changed`

---

## Prerequisites

- ServiceNow admin or Flow Designer author role
- A GitHub Personal Access Token (PAT) with `repo` scope
- The case table must have a field `u_github_issue_number` (String)
- The REST Message `GitHub Repository Dispatch` already created (see `sn-comment-to-github.md` Part 1)
  If you have not done that yet, complete Part 1 of that guide first before continuing here

---

## Part 1 — Add a second HTTP Method to the existing REST Message

You already created the `GitHub Repository Dispatch` REST Message. Now add a second HTTP Method for CR events.

1. Go to: `All > System Web Services > Outbound > REST Messages`
2. Open **GitHub Repository Dispatch**
3. Scroll to the **HTTP Methods** related list
4. Click **New**
5. Fill in:
   - **Name**: `dispatch_cr`
   - **HTTP method**: `POST`
6. Click **Save**

### Add Headers

Add the same three headers as the `dispatch` method:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |
| `Authorization` | `Bearer YOUR_GITHUB_PAT` |

### Add Request Body

In the **HTTP Request** tab, paste this in the **Request body** field:

```json
{"event_type":"servicenow-cr-update","client_payload":{"github_issue_number":"${issue_number}","cr_number":"${cr_number}","cr_sys_id":"${cr_sys_id}","cr_state":"${cr_state}","previous_state":"${previous_state}","cr_environment":"${cr_environment}","case_sys_id":"${case_sys_id}","action":"${action}"}}
```

### Add Variable Substitutions

Click **New** in the **Variable Substitutions** related list for each:

| Name | Test value |
|---|---|
| `issue_number` | `42` |
| `cr_number` | `CHG0012345` |
| `cr_sys_id` | `abc123def456abc123def456abc12345` |
| `cr_state` | `Assess` |
| `previous_state` | `` (empty) |
| `cr_environment` | `Production` |
| `case_sys_id` | `xyz789xyz789xyz789xyz789xyz78901` |
| `action` | `created` |

Click **Update**.

---

## Part 2 — Flow A: CR Created

### 2.1 Open Flow Designer and create the flow

Go to: `All > Process Automation > Flow Designer`

Click **New > Flow**

- **Name**: `SN CR Created → GitHub`
- **Description**: `Notifies GitHub when a Change Request is created from a case`
- **Run as**: `System User`

Click **Submit**.

---

### 2.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Created**
2. **Table**: `Change Request [change_request]`
3. Under **Condition**, click **Add Filters**:
   - Field: `Parent` | Operator: `is not empty`

   This ensures the flow only fires for CRs that are linked to a parent case. Unlinked CRs are ignored.

Click **Done**.

---

### 2.3 Look up the parent Case record

The Change Request record has a `parent` field pointing to the case. You need to look it up to get `u_github_issue_number`.

Click **+** below the trigger.

Search for and select **Look Up Record** (under ServiceNow Core).

Configure it:
- **Table**: `Customer Service Case [sn_customerservice_case]`
- **Filters**:
  - Field: `Sys ID` | Operator: `is` | Value: click the data pill icon → `Trigger > Change Request Record > Parent > Sys ID`

Click **Done**.

This step outputs a full Case record. In later steps it will appear as **Look Up Record > Customer Service Case Record**.

---

### 2.4 Add a Script step to check and build payload

Click **+** below the Look Up Record step.

Select **Script**.

#### Input Variables

Click **+ Create Variable** for each:

| Variable Name | Type | Map to (data pill) |
|---|---|---|
| `github_issue_number` | String | Look Up Record > Customer Service Case Record > `u_github_issue_number` |
| `case_sys_id` | String | Look Up Record > Customer Service Case Record > `Sys ID` |
| `cr_number` | String | Trigger > Change Request Record > `Number` |
| `cr_sys_id` | String | Trigger > Change Request Record > `Sys ID` |
| `cr_state` | String | Trigger > Change Request Record > `State` |
| `cr_environment` | String | Trigger > Change Request Record > `Environment` |

> If there is no `Environment` field on the Change Request in your instance, use whatever field tracks the environment. If it does not exist, skip that variable and hardcode `'Not specified'` in the script.

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseId      = inputs.case_sys_id + '';
  var crNumber    = inputs.cr_number + '';
  var crSysId     = inputs.cr_sys_id + '';
  var crState     = inputs.cr_state + '';
  var crEnv       = inputs.cr_environment + '' || 'Not specified';

  outputs.issue_number   = issueNumber;
  outputs.case_sys_id    = caseId;
  outputs.cr_number      = crNumber;
  outputs.cr_sys_id      = crSysId;
  outputs.cr_state       = crState;
  outputs.cr_environment = crEnv;
  outputs.should_send    = (issueNumber.length > 0 && crNumber.length > 0) ? 'true' : 'false';

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

### 2.5 Add an If Condition

Click **+** below the Script step.

Select **Flow Logic > If**

- Data pill: Script step > `should_send`
- Operator: `is`
- Value: `true`

Click **Done**. Place the next step inside the **then** branch.

---

### 2.6 Add a Script step to call GitHub inside the then branch

Click **+** inside the **then** branch.

Select **Script**.

#### Input Variables

| Variable Name | Type | Map to (data pill) |
|---|---|---|
| `issue_number` | String | Script step (2.4) > `issue_number` |
| `case_sys_id` | String | Script step (2.4) > `case_sys_id` |
| `cr_number` | String | Script step (2.4) > `cr_number` |
| `cr_sys_id` | String | Script step (2.4) > `cr_sys_id` |
| `cr_state` | String | Script step (2.4) > `cr_state` |
| `cr_environment` | String | Script step (2.4) > `cr_environment` |

#### Script

```javascript
(function execute(inputs, outputs) {

  try {
    var rm = new sn_ws.RESTMessageV2('GitHub Repository Dispatch', 'dispatch_cr');

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

    outputs.status  = httpStatus + '';
    outputs.success = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub CR dispatch failed. HTTP ' + httpStatus + ' — ' + response.getBody());
    }

  } catch (ex) {
    outputs.status  = 'error';
    outputs.success = 'false';
    gs.error('GitHub CR dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

#### Output Variables

| Variable Name | Type |
|---|---|
| `status` | String |
| `success` | String |

Click **Done**.

---

### 2.7 Save and Activate Flow A

Click **Save**, then **Activate**.

---

## Part 3 — Flow B: CR State Changed

This is a separate flow. It is almost identical to Flow A but uses a different trigger and includes `previous_state`.

### 3.1 Create the flow

Click **New > Flow**

- **Name**: `SN CR State Changed → GitHub`
- **Description**: `Notifies GitHub when a Change Request state changes`
- **Run as**: `System User`

---

### 3.2 Set the Trigger

Click **Add a trigger**.

1. Select **Record > Updated**
2. **Table**: `Change Request [change_request]`
3. **Condition**:
   - Field: `State` | Operator: `changes`
   - AND Field: `Parent` | Operator: `is not empty`

Click **Done**.

---

### 3.3 Look up the parent Case record

Same as Step 2.3 — add a **Look Up Record** step:

- **Table**: `Customer Service Case [sn_customerservice_case]`
- **Filter**: `Sys ID` is `Trigger > Change Request Record > Parent > Sys ID`

---

### 3.4 Add a Script step to build payload

Same input variables as Step 2.4, plus one additional:

| Variable Name | Type | Map to (data pill) |
|---|---|---|
| `github_issue_number` | String | Look Up Record > Customer Service Case Record > `u_github_issue_number` |
| `case_sys_id` | String | Look Up Record > Customer Service Case Record > `Sys ID` |
| `cr_number` | String | Trigger > Change Request Record > `Number` |
| `cr_sys_id` | String | Trigger > Change Request Record > `Sys ID` |
| `cr_state` | String | Trigger > Change Request Record > `State` |
| `cr_previous_state` | String | Trigger > Change Request Record > `State (Previous value)` |

> "State (Previous value)" is available for updated triggers — it shows what the field was before the update. Look for it under the trigger's data pills.

#### Script

```javascript
(function execute(inputs, outputs) {

  var issueNumber    = inputs.github_issue_number + '';
  var caseId         = inputs.case_sys_id + '';
  var crNumber       = inputs.cr_number + '';
  var crSysId        = inputs.cr_sys_id + '';
  var crState        = inputs.cr_state + '';
  var previousState  = inputs.cr_previous_state + '';

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

Add all the same outputs from 2.4 plus:

| Variable Name | Type |
|---|---|
| `previous_state` | String |

---

### 3.5 Add an If Condition

Same as Step 2.5 — check `should_send` is `true`.

---

### 3.6 Add a Script step to call GitHub

Same structure as Step 2.6 with this script:

```javascript
(function execute(inputs, outputs) {

  try {
    var rm = new sn_ws.RESTMessageV2('GitHub Repository Dispatch', 'dispatch_cr');

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

    outputs.status  = httpStatus + '';
    outputs.success = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub CR state dispatch failed. HTTP ' + httpStatus + ' — ' + response.getBody());
    }

  } catch (ex) {
    outputs.status  = 'error';
    outputs.success = 'false';
    gs.error('GitHub CR state dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

---

### 3.7 Save and Activate Flow B

Click **Save**, then **Activate**.

---

## Testing

### Test Flow A (CR Created)

1. Open a Customer Service Case that has a value in `u_github_issue_number`
2. Click **Create Project Change Request** (top toolbar)
3. Fill in required fields and submit
4. Go to the GitHub issue — within ~30 seconds you should see:

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

### Test Flow B (State Changed)

1. Open an existing Change Request linked to a case with a GitHub issue
2. Change the **State** field to a new value and save
3. Go to the GitHub issue — you should see:

```
Change Request State Updated

CR Number: CHG0012345
State Change: Assess → Implement

---
*Automatically notified by ServiceNow*
```

---

## Checking execution logs

If the GitHub comment does not appear, check the flow logs:

1. Go to: `All > Process Automation > Flow Designer`
2. Click the **Executions** tab at the top
3. Find your flow name and click the execution
4. Expand each step to see its inputs, outputs, and any errors
5. The REST call step will show the HTTP status code returned by GitHub

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow does not fire on CR create | Check the trigger table is `change_request` (not `sn_customerservice_case`) and condition has `Parent is not empty` |
| Look Up Record returns empty | The CR's `parent` field is not set — confirm the CR was created from within the case using the toolbar button |
| `should_send` is false | The parent case has no value in `u_github_issue_number` — set it on the case |
| HTTP 401 | PAT expired or wrong — update the Authorization header in the REST Message |
| HTTP 404 | Wrong org/repo in the REST Message endpoint URL |
| HTTP 422 | The `event_type` value is not registered in the GitHub Actions workflow — confirm `sn-cr-notifier.yml` is deployed to the default branch |
| Previous state is empty | In the trigger data pills, look specifically for "State (Previous value)" not "State" |
