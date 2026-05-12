# ServiceNow CR Notifier → GitHub Flow Designer Guide

When a Change Request is created from a ServiceNow case, or when its state changes, this flow calls the GitHub `repository_dispatch` API to post a notification on the linked GitHub issue.

GitHub Actions file: `.github/workflows/sn-cr-notifier.yml`
Event types sent: `servicenow-cr-update` (action: `created` or `state_changed`)

---

## Prerequisites

- GitHub Personal Access Token (PAT) with `repo` scope stored in ServiceNow as a credential
- The GitHub issue number stored on the parent case (field: `u_github_issue_number`)
- Flow Designer access (All > Process Automation > Flow Designer)
- Connection Alias `GitHub API` already created (see `sn-comment-to-github.md` Step 1 & 2)

---

## Flow A — CR Created

Triggers when a new Change Request is created that is linked to a Customer Service Case.

### Step 1 — Create the Flow

1. Go to **All > Process Automation > Flow Designer**
2. Click **New > Flow**
3. Name: `SN CR Created → GitHub`

### Trigger

- Trigger type: **Record Created**
- Table: `Change Request [change_request]`
- Condition: `Parent` is not empty (ensures it is linked to a case)

### Action 1 — Fetch parent case details

Add a **Look Up Record** action:

| Field | Value |
|---|---|
| Table | `Customer Service Case [sn_customerservice_case]` |
| Filter | `Sys ID` is `trigger.record.parent.sys_id` |

### Action 2 — Script to build payload values

```javascript
(function execute(inputs, outputs) {
  var cr   = inputs.cr_record;
  var case_rec = inputs.case_record;

  outputs.issue_number   = case_rec.u_github_issue_number.toString();
  outputs.cr_number      = cr.number.toString();
  outputs.cr_sys_id      = cr.sys_id.toString();
  outputs.cr_state       = cr.state.getDisplayValue();
  outputs.cr_environment = cr.u_environment.getDisplayValue() || 'Not specified';
  outputs.case_sys_id    = case_rec.sys_id.toString();
  outputs.action         = 'created';
})(inputs, outputs);
```

Add an **If** condition: `issue_number is not empty` — skip if no GitHub issue is linked.

### Action 3 — REST call to GitHub

Add a **REST** step:

| Field | Value |
|---|---|
| Connection Alias | `GitHub API` |
| Base URL | `https://api.github.com` |
| Resource path | `/repos/{owner}/{repo}/dispatches` |
| HTTP Method | `POST` |
| Headers | `Accept: application/vnd.github+json` |

Request body:

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": "<action_2.issue_number>",
    "cr_number": "<action_2.cr_number>",
    "cr_sys_id": "<action_2.cr_sys_id>",
    "cr_state": "<action_2.cr_state>",
    "cr_environment": "<action_2.cr_environment>",
    "case_sys_id": "<action_2.case_sys_id>",
    "action": "created"
  }
}
```

### Step 2 — Activate

Click **Activate**. Test by creating a Change Request linked to a case that has a GitHub issue number.

---

## Flow B — CR State Changed

Triggers when the state of an existing Change Request changes.

### Step 1 — Create the Flow

1. Click **New > Flow**
2. Name: `SN CR State Changed → GitHub`

### Trigger

- Trigger type: **Record Updated**
- Table: `Change Request [change_request]`
- Condition: `State` changes AND `Parent` is not empty

### Action 1 — Fetch parent case details

Same **Look Up Record** action as Flow A.

### Action 2 — Script to build payload values

```javascript
(function execute(inputs, outputs) {
  var cr       = inputs.cr_record;
  var case_rec = inputs.case_record;

  outputs.issue_number   = case_rec.u_github_issue_number.toString();
  outputs.cr_number      = cr.number.toString();
  outputs.cr_sys_id      = cr.sys_id.toString();
  outputs.cr_state       = cr.state.getDisplayValue();
  outputs.previous_state = cr.state.changesFrom() || '';
  outputs.case_sys_id    = case_rec.sys_id.toString();
})(inputs, outputs);
```

### Action 3 — REST call to GitHub

Same **REST** step as Flow A, with this request body:

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": "<action_2.issue_number>",
    "cr_number": "<action_2.cr_number>",
    "cr_sys_id": "<action_2.cr_sys_id>",
    "cr_state": "<action_2.cr_state>",
    "previous_state": "<action_2.previous_state>",
    "case_sys_id": "<action_2.case_sys_id>",
    "action": "state_changed"
  }
}
```

### Step 2 — Activate

Click **Activate**. Test by moving a CR linked to a case from one state to another.

---

## What the GitHub side produces

**On CR created:**
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

**On state change:**
```
Change Request State Updated

CR Number: CHG0012345
State Change: Assess → Implement

---
*Automatically notified by ServiceNow*
```

> Once a CR is created, the GitHub issue → ServiceNow field update sync is automatically blocked. Comments continue to sync both ways.
