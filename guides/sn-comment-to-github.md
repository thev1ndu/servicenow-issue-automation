# ServiceNow Comment → GitHub Flow Designer Guide

When an agent adds an additional comment or work note to a ServiceNow case, this flow calls the GitHub `repository_dispatch` API to post it as a comment on the linked GitHub issue.

GitHub Actions file: `.github/workflows/sn-comment-to-github.yml`
Event type sent: `servicenow-note`

---

## Prerequisites

- GitHub Personal Access Token (PAT) with `repo` scope stored in ServiceNow as a credential
- The GitHub issue number stored on the case (field: `u_github_issue_number`)
- Flow Designer access (All > Process Automation > Flow Designer)

---

## Step 1 — Create a Credential Alias for GitHub

1. Go to **All > Connections & Credentials > Credentials**
2. Click **New**, select **Basic Auth Credentials**
3. Fill in:
   - Name: `GitHub PAT`
   - User name: your GitHub username
   - Password: your GitHub Personal Access Token
4. Save

---

## Step 2 — Create a Connection Alias

1. Go to **All > Connections & Credentials > Connection & Credential Aliases**
2. Click **New**
3. Fill in:
   - Name: `GitHub API`
   - Type: `Connection and Credential`
   - Connection URL: `https://api.github.com`
4. Under **Credentials**, link to the `GitHub PAT` credential created above
5. Save

---

## Step 3 — Create the Flow

1. Go to **All > Process Automation > Flow Designer**
2. Click **New > Flow**
3. Name: `SN Comment to GitHub`
4. Description: `Posts ServiceNow case comments to the linked GitHub issue`

### Trigger

- Trigger type: **Record Updated**
- Table: `Customer Service Case [sn_customerservice_case]`
- Condition: `Additional comments` changes **OR** `Work notes` changes

> Use two separate conditions joined with OR, or create two separate flows — one per field.

### Action 1 — Evaluate which field changed

Add a **Script** action to determine the note type and content:

```javascript
(function execute(inputs, outputs) {
  var gr = inputs.record;
  var noteText = '';
  var noteType = '';

  // Check which journal field was just updated
  var journal = new GlideRecordUtil();
  var addComments = gr.comments.getJournalEntry(1);
  var workNotes   = gr.work_notes.getJournalEntry(1);

  if (addComments) {
    noteText = addComments;
    noteType = 'additional_comments';
  } else if (workNotes) {
    noteText = workNotes;
    noteType = 'work_notes';
  }

  outputs.note_text  = noteText;
  outputs.note_type  = noteType;
  outputs.sn_user    = gs.getUserDisplayName();
  outputs.issue_num  = gr.u_github_issue_number.toString();
  outputs.case_num   = gr.number.toString();
})(inputs, outputs);
```

Define outputs: `note_text` (String), `note_type` (String), `sn_user` (String), `issue_num` (String), `case_num` (String)

Add an **If** condition: `note_text is not empty` — only continue if there is actual content.

### Action 2 — REST call to GitHub

Add a **REST** step:

| Field | Value |
|---|---|
| Connection Alias | `GitHub API` |
| Base URL | `https://api.github.com` |
| Resource path | `/repos/{owner}/{repo}/dispatches` |
| HTTP Method | `POST` |
| Headers | `Accept: application/vnd.github+json` |
| Request body | See below |

Request body (use Script or Template):

```json
{
  "event_type": "servicenow-note",
  "client_payload": {
    "issue_number": "<action_1.issue_num>",
    "note_text": "<action_1.note_text>",
    "note_type": "<action_1.note_type>",
    "case_number": "<action_1.case_num>",
    "sn_user": "<action_1.sn_user>"
  }
}
```

Replace `{owner}` and `{repo}` with your GitHub organisation and repository name.

---

## Step 4 — Activate the Flow

1. Click **Activate** in the top right
2. Test by adding a comment to a case that has a GitHub issue number set

---

## What the GitHub side produces

The `sn-comment-to-github.yml` workflow posts a comment on the issue like:

```
💬 ServiceNow Comment — Case CS0439913

Agent reply text here

---
*Posted by John Smith via ServiceNow*
```

For work notes the label changes to `📋 ServiceNow Work Note`.
