# Guide: ServiceNow Comment → GitHub (Flow Designer)

When an agent adds an additional comment to a ServiceNow case, this flow
posts it as a comment on the linked GitHub issue.

GitHub Actions file that receives it: `.github/workflows/sn-comment-to-github.yml`
GitHub event type sent: `servicenow-note`

---

## What happens end to end

1. Agent types a comment in the ServiceNow case and clicks Post
2. This Flow fires because the `Additional comments` field changed
3. A Script step reads the latest comment and the linked GitHub issue number
4. A REST call is made to the GitHub API (`repository_dispatch`) using the `dispatch` HTTP Method
5. The `sn-comment-to-github.yml` GitHub Action picks it up and posts the comment on the issue

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
- **Name**: `SN Comment to GitHub`
- **Description**: `Posts ServiceNow case additional comments to the linked GitHub issue`
- **Run as**: `System User`

Click **Submit**. The flow editor opens.

---

### 1.2 Set the Trigger

Click **Add a trigger** (the first block at the top of the canvas).

1. Select **Record > Updated**
2. **Table**: type and select `Customer Service Case [sn_customerservice_case]`
3. Under **Condition**, click **Add Filters**:
   - Field: `Additional comments` | Operator: `changes`
4. Leave **Run trigger** as: `For every update that causes condition to be true`

Click **Done**.

---

### 1.3 Add Script Step 1 — Read the comment and prepare data

Click the **+** (plus) button below the trigger.

In the action picker, search for **Script** and select it.

A Script step block appears on the canvas. Click it to open the configuration panel on the right.

#### Define Input Variables

Inside the Script step panel, find **Input Variables**. Click **+ Create Variable** for each row below.

For each variable:
1. Enter the **Name**
2. Set the **Type** to `String`
3. Click the **data pill icon** (circular icon at the right of the value field) to open the data picker
4. In the data picker, expand **Trigger > Customer Service Case Record** and select the matching field

| Variable Name | Type | Data pill to select |
|---|---|---|
| `github_issue_number` | String | Trigger > Customer Service Case Record > **u_github_issue_number** |
| `case_number` | String | Trigger > Customer Service Case Record > **Number** |
| `case_sys_id` | String | Trigger > Customer Service Case Record > **Sys ID** |
| `account_name` | String | Trigger > Customer Service Case Record > **Account** > **Name** |

> Do NOT map `Additional comments` as a data pill — journal fields return a Java object
> reference (e.g. `com.glide.glideobject.Journal@6e3f83c9`) instead of the actual text.
> The script reads the journal entry directly using `GlideRecord` instead.

#### Write the Script

In the **Script** area, paste:

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseNumber  = inputs.case_number + '';
  var caseSysId   = inputs.case_sys_id + '';
  var accountName = inputs.account_name + '';

  // Journal fields must be read via GlideRecord — data pills return a Java object reference
  var gr = new GlideRecord('sn_customerservice_case');
  gr.get(caseSysId);
  var commentText = gr.comments.getJournalEntry(1) + '';

  outputs.issue_number  = issueNumber;
  outputs.note_text     = commentText;
  outputs.note_type     = 'additional_comments';
  outputs.case_number   = caseNumber;
  outputs.case_sys_id   = caseSysId;
  outputs.sn_user       = gs.getUserDisplayName();
  outputs.account_name  = accountName;

  // Only send if the case has a GitHub issue linked AND a comment was actually posted
  outputs.should_send = (issueNumber.length > 0 && commentText.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

> `getJournalEntry(1)` returns the single most recent entry added in this update — exactly
> what the agent just typed. Passing `1` means "get the latest 1 entry".

#### Define Output Variables

In the **Output Variables** section below the script editor, click **+ Create Variable** for each:

| Variable Name | Type |
|---|---|
| `issue_number` | String |
| `note_text` | String |
| `note_type` | String |
| `case_number` | String |
| `case_sys_id` | String |
| `sn_user` | String |
| `account_name` | String |
| `should_send` | String |

Click **Done** to close the Script step panel.

---

### 1.4 Add an If Condition — skip if nothing to send

Click the **+** below the Script step.

Select **Flow Logic > If**

Configure:
- Click the data pill icon next to the condition field
- Select: **Script step 1 > `should_send`**
- Operator: `is`
- Value: type `true`

Click **Done**.

This creates two branches. All remaining steps go inside the **then** branch (the left/true side).

---

### 1.5 Add Script Step 2 — Call the GitHub REST Message

Click **+** inside the **then** branch.

Select **Script**.

#### Input Variables

Map each from the previous Script step's outputs:

| Variable Name | Type | Data pill to select |
|---|---|---|
| `issue_number` | String | Script step 1 > `issue_number` |
| `note_text` | String | Script step 1 > `note_text` |
| `note_type` | String | Script step 1 > `note_type` |
| `case_number` | String | Script step 1 > `case_number` |
| `case_sys_id` | String | Script step 1 > `case_sys_id` |
| `sn_user` | String | Script step 1 > `sn_user` |
| `account_name` | String | Script step 1 > `account_name` |

#### Script

```javascript
(function execute(inputs, outputs) {

  try {
    // Read GitHub config from the system property
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

    // 'GitHub Integration' = REST Message name, 'dispatch' = HTTP Method name
    var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');

    // Override endpoint and auth using values from the system property
    rm.setEndpoint(endpoint);
    rm.setRequestHeader('Authorization', 'token ' + config.token);

    // Set the request body directly — no variable substitutions needed
    rm.setRequestBody(JSON.stringify({
      event_type: 'servicenow-note',
      client_payload: {
        issue_number: inputs.issue_number,
        note_text:    inputs.note_text,
        note_type:    inputs.note_type,
        case_number:  inputs.case_number,
        case_sys_id:  inputs.case_sys_id,
        sn_user:      inputs.sn_user
      }
    }));

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.http_status = httpStatus + '';
    outputs.success     = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub dispatch failed. HTTP ' + httpStatus + ' Body: ' + response.getBody());
    }

  } catch (ex) {
    outputs.http_status = 'error';
    outputs.success     = 'false';
    gs.error('GitHub dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

> `gs.getProperty('github.dispatch.config')` reads the keyed config and selects the entry for `REPO`.
> `rm.setEndpoint()` overrides the placeholder URL on the REST Message with the real owner/repo URL.
> `rm.setRequestHeader('Authorization', ...)` injects the PAT token as a bearer token header.
> `rm.setRequestBody()` sets the JSON body directly — no `${placeholder}` substitutions needed.
> GitHub returns HTTP `204` on a successful `repository_dispatch` call.

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

The flow is now live.

---

## Testing

1. Open any Customer Service Case that has a value in `u_github_issue_number`
2. Scroll to **Additional comments (Customer visible)**
3. Type a test message and click **Post**
4. Go to the linked GitHub issue — the comment should appear within ~30 seconds:

```
💬 ServiceNow Comment — Case CS0000001

Test message here

---
*Posted by John Smith via ServiceNow*
```

---

## Checking execution logs

If the GitHub comment does not appear:

1. Go to: `All > Process Automation > Flow Designer`
2. Click the **Executions** tab at the top
3. Find `SN Comment to GitHub` and click the execution row
4. Expand each step — the Script step 2 output will show the `http_status` returned by GitHub

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow does not trigger | Confirm the trigger condition is `Additional comments changes` |
| `should_send` is `false` | The case has no value in `u_github_issue_number` — fill that field on the case |
| `http_status` is `401` | The token in `github.dispatch.config` is expired or wrong |
| `http_status` is `404` | The owner or repo in `github.dispatch.config` is wrong |
| `http_status` is `422` | The `sn-comment-to-github.yml` workflow is not deployed to the default branch |
| Comment shows `Journal@...` on GitHub | The old flow used a data pill for `Additional comments` — replace `comments_changed` input with `case_sys_id` and use `gr.comments.getJournalEntry(1)` in the script |
| Comment is blank on GitHub | `getJournalEntry(1)` returned empty — confirm the trigger fired on an actual comment update, not a field edit |
