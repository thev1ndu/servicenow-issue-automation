# How this works (end-to-end)

This system is a **GitHub-native** automation that:

- **Creates a ServiceNow case from a GitHub issue** (when the issue is labeled correctly).
- **Posts ServiceNow Change Request (CR) updates back to the GitHub issue** (when ServiceNow sends a webhook-style `repository_dispatch`).

It’s designed to keep the **ServiceNow-side changes minimal**: ServiceNow only needs to send **`repository_dispatch`** events to GitHub when something happens (CR created / CR state changed).

---

## Environments and responsibilities

### GitHub (this repository)

- **Input 1 (Issue labels)**: A human adds `SRType/*` and `CatalogueItem/*` labels on an issue created from an SR template.
- **Input 2 (ServiceNow events)**: ServiceNow calls GitHub’s REST API `repository_dispatch` when a CR is created or changes state.
- **Output**: GitHub Actions posts comments on the issue and calls ServiceNow (for case creation).

Workflows:

- `./.github/workflows/issue-servicenow.yml`
  - Trigger: `issues:labeled`
  - Does: validate template → create ServiceNow case → comment “watching for CRs”
- `./.github/workflows/servicenow-create-case.yml`
  - Trigger: `workflow_call` (reusable)
  - Does: build payload from issue template fields → POST to ServiceNow → comment success/failure
- `./.github/workflows/servicenow-inbound.yml`
  - Trigger: `repository_dispatch`
  - Does: handle ServiceNow→GitHub events (CR updates, optional case creation dispatch)

### ServiceNow

- **Creates/updates cases and change requests**
- **Sends outbound events to GitHub** using `sn_ws.RESTMessageV2` (Flow Designer “Run Script”, Script Include, or Scripted REST resource that forwards outbound).

---

## Main flow: GitHub Issue → ServiceNow case

### What the user does

1. Create an issue using the **SR issue template** (see `.github/ISSUE_TEMPLATE/sr-generic.md`).
2. Add both labels:
   - `SRType/<something>` (example: `SRType/Normal Change`)
   - `CatalogueItem/<something>` (example: `CatalogueItem/Generic Requests`)

### What GitHub Actions does

`issue-servicenow.yml` runs:

1. **Gate**
   - Skips if the label added is not an SR label.
   - Skips if a “ServiceNow Case Created Successfully” comment already exists.

2. **Validate**
   - Confirms the issue body contains an SR **Template Marker** (e.g. `SR Generic`).
   - Confirms both required labels are present.
   - Confirms title starts with **`[SR-Change]:`**.
   - Confirms required sections exist for the selected `CatalogueItem/*`.
   - Enforces catalog-specific rules (example: Request Logs requires **Priority = Critical**).
   - On success:
     - Adds label **`validation-passed`**
     - Posts “✅ Template Validation Passed”
   - On failure:
     - Removes SR-related labels
     - Posts “❌ Template Validation Failed”

3. **Create ServiceNow case**
   - Calls reusable workflow `servicenow-create-case.yml`
   - Extracts all `###` sections and converts them into a JSON payload
   - Sends it to ServiceNow using GitHub secrets for authentication
   - Posts “✅ ServiceNow Case Created Successfully” with the clickable case link

4. **CR watcher comment**
   - Posts “👀 Watching for Change Requests”
   - This does **not** poll ServiceNow; it just tells users that CR updates will arrive asynchronously.

---

## Inbound flow: ServiceNow CR → GitHub Issue comment

### What happens in ServiceNow

When a **Change Request is created** or its **state changes**, ServiceNow sends an outbound HTTP request to GitHub:

- **HTTP**: `POST https://api.github.com/repos/<OWNER>/<REPO>/dispatches`
- **Body**:

```json
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": 42,
    "action": "created",
    "cr_number": "CHG0038376",
    "cr_sys_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "cr_state": "New",
    "cr_environment": "Development",
    "case_sys_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb"
  }
}
```

### What GitHub Actions does

`servicenow-inbound.yml` runs job **`cr-update`** and:

- Validates required fields:
  - `github_issue_number`
  - `cr_number`
  - `action` (`created` or `state_changed`)
- Builds a ServiceNow UI link using `SERVICENOW_UI_URL` and `cr_sys_id`
- On `action=created`:
  - Tries to query ServiceNow for **all CRs linked to the case** using `case_sys_id`
  - Posts a **new comment** like:
    - “New Change Request Created”
    - CR number (clickable)
    - State + Environment
    - “All Change Requests for this Case (N)” list
    - Footer “Automatically notified by ServiceNow”
- On `action=state_changed`:
  - Posts a **new comment** like:
    - “Change Request State Updated”
    - State change: previous → current
    - Footer “Automatically notified by ServiceNow”

---

## Does a Case work note become a GitHub comment?

**Not by default in the current implementation.**

Right now, GitHub is updated when ServiceNow sends **CR events** (`servicenow-cr-update`).

If you want: **“when a user adds a new work note on the ServiceNow case → create a GitHub issue comment”**, you need one extra ServiceNow outbound event + a small GitHub handler:

### Required addition (ServiceNow)

- Create a Flow / Business Rule on the **case table** (e.g. `sn_customerservice_case`):
  - Trigger when **work notes** changes
  - Send a `repository_dispatch` with a new event type, for example: `servicenow-case-worknote`
  - Include `github_issue_number`, `case_number`, `case_sys_id`, and the work note text

Example payload:

```json
{
  "event_type": "servicenow-case-worknote",
  "client_payload": {
    "github_issue_number": 42,
    "case_number": "CS0012345",
    "case_sys_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "work_note": "We are investigating the root cause.",
    "author": "john.doe",
    "visibility": "internal"
  }
}
```

### Required addition (GitHub)

- Extend `servicenow-inbound.yml` to accept that new `event_type` and post it as an issue comment.

If you want this behavior, tell me which case table you’re using and whether you want:

- **Work notes only** or also **additional comments**
- Post **every update** or only when state/assignment changes
- Include author + timestamp or keep it minimal

---

## What you’ll see in GitHub

### After validation

- “✅ Template Validation Passed”

### After case creation

- “✅ ServiceNow Case Created Successfully” with the ServiceNow case link

### After CR created (from ServiceNow)

- “New Change Request Created”
- CR number + state + environment
- “All Change Requests for this Case (N)”
- “Automatically notified by ServiceNow”

### After CR state changes (from ServiceNow)

- “Change Request State Updated”
- “State Change: <previous> → <current>”
- “Automatically notified by ServiceNow”

