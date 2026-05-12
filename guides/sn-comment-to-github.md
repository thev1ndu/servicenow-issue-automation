# Guide: ServiceNow Comment → GitHub (Flow Designer)

When an agent adds an additional comment or work note to a ServiceNow case, this flow
posts it as a comment on the linked GitHub issue.

GitHub Actions file that receives it: `.github/workflows/sn-comment-to-github.yml`
GitHub event type sent: `servicenow-note`

---

## What happens end to end

1. Agent types a comment in the ServiceNow case and clicks Post
2. This Flow fires because the `comments` or `work_notes` field changed
3. A Script step reads the latest journal entry and the linked GitHub issue number
4. A REST call is made to the GitHub API (`repository_dispatch`)
5. The `sn-comment-to-github.yml` GitHub Action picks it up and posts the comment on the issue

---

## Prerequisites

- ServiceNow admin or Flow Designer author role
- A GitHub Personal Access Token (PAT) with `repo` scope
- The case table must have a field `u_github_issue_number` (String) that stores the GitHub issue number

---

## Part 1 — Create a REST Message for GitHub (do this once, shared with both flows)

This creates a reusable outbound REST definition that both flows will use.

### 1.1 Create the REST Message record

1. In the ServiceNow top navigation, type **Outbound REST Messages** or go to:
   `All > System Web Services > Outbound > REST Messages`
2. Click **New**
3. Fill in these fields:
   - **Name**: `GitHub Repository Dispatch`
   - **Description**: `Sends repository_dispatch events to GitHub Actions`
   - **Endpoint**: `https://api.github.com/repos/YOUR_ORG/YOUR_REPO/dispatches`
     Replace `YOUR_ORG` and `YOUR_REPO` with your actual GitHub org and repo name
   - **Authentication type**: `No authentication` (we will pass the token manually in the header)
4. Click **Submit** to save

### 1.2 Add the HTTP Method

After saving, you will be inside the REST Message record.

1. Scroll down to the **HTTP Methods** related list
2. Click **New**
3. Fill in:
   - **Name**: `dispatch`
   - **HTTP method**: `POST`
   - **Endpoint**: leave blank (inherits from the REST Message)
4. Click **Save** (not Submit yet — stay on this record)

### 1.3 Add HTTP Request Headers

Still on the HTTP Method record, scroll to **HTTP Request** tab or the **HTTP Headers** section.

Click **Insert a new row** and add these two headers:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |
| `Authorization` | `Bearer YOUR_GITHUB_PAT` |

Replace `YOUR_GITHUB_PAT` with your actual Personal Access Token.

> **Security tip:** Store the PAT as a ServiceNow System Property or Connection Credential and reference it using `${gs.getProperty('github.pat')}` instead of pasting it directly.

### 1.4 Add the Request Body

Still on the HTTP Method record, click the **HTTP Request** tab.

In the **Request body** field, paste:

```json
{"event_type":"servicenow-note","client_payload":{"issue_number":"${issue_number}","note_text":"${note_text}","note_type":"${note_type}","case_number":"${case_number}","sn_user":"${sn_user}"}}
```

These `${variable_name}` placeholders are ServiceNow REST Message variables. They will be filled in by the script before the call is made.

### 1.5 Add REST Message Variables

Scroll down to the **Variable Substitutions** related list (or the **Variables** tab).

Click **New** for each of the following:

| Name | Test value (example) |
|---|---|
| `issue_number` | `42` |
| `note_text` | `Test comment from ServiceNow` |
| `note_type` | `additional_comments` |
| `case_number` | `CS0000001` |
| `sn_user` | `John Smith` |

Click **Update** after adding all variables.

---

## Part 2 — Create the Flow

### 2.1 Open Flow Designer

Go to: `All > Process Automation > Flow Designer`

Click **New > Flow**

Fill in:
- **Name**: `SN Comment to GitHub`
- **Description**: `Posts ServiceNow case comments and work notes to the linked GitHub issue`
- **Run as**: `System User` (so it has access to case fields)

Click **Submit**. The flow editor opens.

---

### 2.2 Set the Trigger

Click **Add a trigger** (the first block at the top of the canvas).

1. Select **Record > Updated**
2. In the **Table** field, type and select: `Customer Service Case [sn_customerservice_case]`
3. Under **Condition**, click **Add Filters**:
   - Field: `Additional comments` | Operator: `changes`
   - Click **OR**
   - Field: `Work notes` | Operator: `changes`
4. Leave **Run trigger** set to: `For every update that causes condition to be true`

Click **Done**.

---

### 2.3 Add a Script Step

Click the **+** (plus) button below the trigger to add an action.

1. In the action picker, search for **Script** and select it
2. A Script step block appears. Click on it to configure it.

#### Define Input Variables for the Script step

Inside the Script step panel on the right, find the **Input Variables** section.

Click **+ Create Variable** for each of the following. For each one, set the **Name** and **Type**, then click the **data pill icon** (looks like a small circle/dot) to map it to a value from the trigger:

| Variable Name | Type | Map to (data pill) |
|---|---|---|
| `github_issue_number` | String | Trigger > Customer Service Case Record > `u_github_issue_number` |
| `case_number` | String | Trigger > Customer Service Case Record > `Number` |
| `comments_changed` | String | Trigger > Customer Service Case Record > `Additional comments` |
| `work_notes_changed` | String | Trigger > Customer Service Case Record > `Work notes` |

> To pick the data pill: after setting the Type, click the circular icon at the right end of the value field. A panel opens showing the Trigger record fields. Expand "Customer Service Case Record" and click the field you want.

#### Write the Script

In the **Script** area of the step, paste the following:

```javascript
(function execute(inputs, outputs) {

  var issueNumber = inputs.github_issue_number + '';
  var caseNumber  = inputs.case_number + '';

  // Determine which journal field was updated and get the latest entry
  var noteText = '';
  var noteType = '';

  // getJournalEntry(1) returns the most recent entry added in this update
  var latestComment  = inputs.comments_changed;
  var latestWorkNote = inputs.work_notes_changed;

  if (latestComment && latestComment.length > 0) {
    noteText = latestComment;
    noteType = 'additional_comments';
  } else if (latestWorkNote && latestWorkNote.length > 0) {
    noteText = latestWorkNote;
    noteType = 'work_notes';
  }

  outputs.issue_number = issueNumber;
  outputs.note_text    = noteText;
  outputs.note_type    = noteType;
  outputs.case_number  = caseNumber;
  outputs.sn_user      = gs.getUserDisplayName();
  outputs.should_send  = (issueNumber.length > 0 && noteText.length > 0) ? 'true' : 'false';

})(inputs, outputs);
```

#### Define Output Variables for the Script step

In the **Output Variables** section below the script editor, click **+ Create Variable** for each:

| Variable Name | Type |
|---|---|
| `issue_number` | String |
| `note_text` | String |
| `note_type` | String |
| `case_number` | String |
| `sn_user` | String |
| `should_send` | String |

Click **Done** on the Script step.

---

### 2.4 Add an If Condition (skip if no GitHub issue linked)

Click the **+** below the Script step.

Select **Flow Logic > If**

Configure the condition:
- Data pill: click the pill icon, select **Script step > `should_send`**
- Operator: `is`
- Value: `true`

Click **Done**. This creates a branch. Place the next step inside the **then** branch.

---

### 2.5 Add a Script Step to Call GitHub

Click **+** inside the **then** branch of the If condition.

Select **Script** again.

#### Input Variables

| Variable Name | Type | Map to (data pill) |
|---|---|---|
| `issue_number` | String | Previous Script step > `issue_number` |
| `note_text` | String | Previous Script step > `note_text` |
| `note_type` | String | Previous Script step > `note_type` |
| `case_number` | String | Previous Script step > `case_number` |
| `sn_user` | String | Previous Script step > `sn_user` |

#### Script

```javascript
(function execute(inputs, outputs) {

  try {
    var rm = new sn_ws.RESTMessageV2('GitHub Repository Dispatch', 'dispatch');

    rm.setStringParameterNoEscape('issue_number', inputs.issue_number);
    rm.setStringParameterNoEscape('note_text',    inputs.note_text);
    rm.setStringParameterNoEscape('note_type',    inputs.note_type);
    rm.setStringParameterNoEscape('case_number',  inputs.case_number);
    rm.setStringParameterNoEscape('sn_user',      inputs.sn_user);

    var response   = rm.execute();
    var httpStatus = response.getStatusCode();

    outputs.status  = httpStatus + '';
    outputs.success = (httpStatus == 204 || httpStatus == 200) ? 'true' : 'false';

    if (outputs.success == 'false') {
      gs.warn('GitHub dispatch failed. HTTP ' + httpStatus + ' — ' + response.getBody());
    }

  } catch (ex) {
    outputs.status  = 'error';
    outputs.success = 'false';
    gs.error('GitHub dispatch exception: ' + ex.message);
  }

})(inputs, outputs);
```

> `sn_ws.RESTMessageV2` is the ServiceNow class for calling an Outbound REST Message by name.
> `setStringParameterNoEscape` fills in the `${variable}` placeholders in the request body without HTML encoding.
> GitHub's `repository_dispatch` returns HTTP 204 on success.

#### Output Variables

| Variable Name | Type |
|---|---|
| `status` | String |
| `success` | String |

Click **Done**.

---

### 2.6 Save and Activate

1. Click **Save** (top right)
2. Click **Activate**

The flow is now live.

---

## Testing

1. Open any Customer Service Case that has a GitHub issue number in `u_github_issue_number`
2. Scroll to the **Additional comments (Customer visible)** field
3. Type a test message and click **Post**
4. Go to the linked GitHub issue — the comment should appear within ~30 seconds as:

```
💬 ServiceNow Comment — Case CS0000001

Test message here

---
*Posted by John Smith via ServiceNow*
```

---

## Troubleshooting

| Problem | What to check |
|---|---|
| Flow does not trigger | Check the condition — make sure `comments changes` OR `work_notes changes` is set correctly |
| HTTP 401 from GitHub | The PAT in the Authorization header is wrong or expired |
| HTTP 404 from GitHub | The org/repo in the REST Message endpoint URL is incorrect |
| `should_send` is false | The case has no value in `u_github_issue_number` — fill that field first |
| Comment appears blank on GitHub | Check the data pill mapping for `comments_changed` — it must map to the case's Additional comments field |

To see flow execution logs: `All > Process Automation > Flow Designer > Executions` tab.
