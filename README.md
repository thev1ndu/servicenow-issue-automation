# GitHub Actions Implementation

This document covers every file introduced on the GitHub side: workflows, issue templates, label definitions, and configuration files. Nothing here touches ServiceNow directly — GitHub Actions calls the ServiceNow Scripted REST API and receives events via `repository_dispatch`.

---

## Repository Files Introduced

| Path | Type | Purpose |
|------|------|---------|
| `.github/workflows/issue-servicenow.yml` | Workflow | Validates issues and orchestrates case creation |
| `.github/workflows/servicenow-create-case.yml` | Reusable workflow | Builds the payload and POSTs to ServiceNow |
| `.github/workflows/servicenow-update-case.yml` | Reusable workflow | PATCHes the case when an issue is edited |
| `.github/workflows/servicenow-inbound.yml` | Workflow | Handles `validation-passed` repository_dispatch events |
| `.github/workflows/sn-cr-notifier.yml` | Workflow | Handles `servicenow-cr-update` events from ServiceNow |
| `.github/workflows/github-comment-to-sn.yml` | Workflow | Syncs human GitHub comments to the ServiceNow case |
| `.github/workflows/sn-comment-to-github.yml` | Workflow | Handles `servicenow-note` events from ServiceNow |
| `.github/ISSUE_TEMPLATE/sr-generic.md` | Issue template | SR Generic Requests — full 14-field form |
| `.github/ISSUE_TEMPLATE/sr-request-logs.md` | Issue template | SR Request Logs — log requests, Critical priority only |
| `.github/ISSUE_TEMPLATE/sr-standard-generic.md` | Issue template | SR Standard Generic — simplified 5-field form |
| `.github/ISSUE_TEMPLATE/sr-information.md` | Issue template | SR Information — minimal 3-field form |
| `.github/ISSUE_TEMPLATE/config.yml` | Template config | Disables blank issue creation |
| `.github/labels.yml` | Label definitions | Defines all `SRType/*`, `CatalogueItem/*`, and `validation-passed` labels |
| `.github/servicenow-config.yml` | Static config | ServiceNow constants applied to every case (account, project, product, etc.) |

---

## GitHub Secrets Required

These four secrets must be configured in the repository before any workflow can run:

| Secret | Value |
|--------|-------|
| `SERVICENOW_URL` | Full Scripted REST endpoint: `https://<instance>.service-now.com/api/<scope>/github_case/v1/case` |
| `SERVICENOW_UI_URL` | Base instance URL: `https://<instance>.service-now.com` — used to build clickable case and CR links |
| `SERVICENOW_USERNAME` | ServiceNow user with REST API access |
| `SERVICENOW_PASSWORD` | Password for the above user |

---

## Labels

Labels drive the validation and routing logic. They are defined in `.github/labels.yml` and must exist in the repository before workflows run.

| Label | Colour | Role |
|-------|--------|------|
| `SRType/Normal Change` | Blue | Identifies the SR change type |
| `SRType/Standard Change` | Blue | Identifies the SR change type |
| `SRType/Emergency Change` | Red | Identifies the SR change type (high priority) |
| `CatalogueItem/Generic Requests` | Yellow | Selects the Generic Requests validation ruleset |
| `CatalogueItem/Request Logs` | Yellow | Selects the Request Logs validation ruleset |
| `CatalogueItem/Standard Generic` | Yellow | Selects the Standard Generic validation ruleset |
| `CatalogueItem/Information` | Yellow | Selects the Information validation ruleset |
| `validation-passed` | Green | Added by automation after successful validation |

The main workflow reads `SRType/*` and `CatalogueItem/*` labels from the issue to determine which field-validation ruleset to apply. Exactly one label from each group must be present.

---

## Static Config: `.github/servicenow-config.yml`

This YAML file holds ServiceNow constants that are applied to **every** case created from GitHub. Edit once, commit — no secrets or repository variables needed.

```yaml
case:
  category: "Issue"
  account: "Your Account Name"
  announcement_type: "General"

project:
  name: "Your Project Name"
  product: "Your Product Name"
  wso2_product: "Your WSO2 Product Name"
```

All values must **exactly** match the display names of the corresponding records in ServiceNow. For reference fields (`account`, `project`, `product`, `wso2_product`), a name mismatch causes ServiceNow to silently leave the field blank — no error is returned.

---

## Issue Templates

Four templates are provided under `.github/ISSUE_TEMPLATE/`. `config.yml` disables blank issues so users must always pick a template.

### `sr-generic.md` — SR Generic Requests

The most comprehensive form. Validation requires all 14 sections to be filled in.

Required sections: Short Description, Description, Priority, Impact, Impact Description (Overall), Impact Description (Customer), Environment Details, Affected Component, Affected Services, Service Outage/Downtime, Is a maintenance window required, Implementation Plan, Test Plan, Monitoring Checks.

### `sr-request-logs.md` — SR Request Logs

For log retrieval requests. Validation additionally enforces that Priority is set to `Critical`.

Required sections: Short Description, Description, Priority, Impact, Impact Description, Customer Project, Environment Details.

### `sr-standard-generic.md` — SR Standard Generic

A lighter form for routine requests.

Required sections: Short Description, Description, Priority, Impact, Environment Details.

### `sr-information.md` — SR Information

Minimal form for information-only requests.

Required sections: Request Description, Impact, Customer Project.

---

## Workflow: `issue-servicenow.yml` (Main Orchestrator)

**Trigger:** `issues: [opened, edited, labeled]`

This is the entry point for all inbound case operations. It runs every time an issue is created, edited, or labeled and decides whether to validate and create/update a ServiceNow case.

**Logic flow:**

```
Issue event fires
       ↓
Gate: Does the issue body contain a known SR template marker?
  → No: exit silently (not an SR issue)
       ↓
Check for existing SRType/* and CatalogueItem/* labels
  → Missing: exit silently (waiting for labels to be added)
       ↓
Validate issue title starts with [SR-Change]:
  → Fail: post failure comment, exit
       ↓
Validate all required fields per CatalogueItem/* ruleset
  → Fail: remove SR labels, post failure comment, exit
       ↓
Add validation-passed label
Post validation success comment
       ↓
Case already exists? (check for "ServiceNow Case Created Successfully" comment)
  → Yes + issue was edited + no CRs yet: call servicenow-update-case.yml
  → No: call servicenow-create-case.yml
```

**Validation rules per catalog item:**

| CatalogueItem | Fields that must not be blank | Extra rule |
|---------------|------------------------------|------------|
| Generic Requests | Short Description, Description, Priority, Impact, Impact Description (Overall), Impact Description (Customer), Environment Details, Affected Component, Affected Services, Service Outage/Downtime, Is a maintenance window required, Implementation Plan, Test Plan, Monitoring Checks | — |
| Request Logs | Short Description, Description, Priority, Impact, Impact Description, Customer Project, Environment Details | Priority must be "Critical" |
| Standard Generic | Short Description, Description, Priority, Impact, Environment Details | — |
| Information | Request Description, Impact, Customer Project | — |

A field is considered blank if its value is `_No response_` (the GitHub template default) or empty.

---

## Workflow: `servicenow-create-case.yml` (Reusable)

**Trigger:** `workflow_call` (called by `issue-servicenow.yml`)

Builds the full JSON payload from the issue body and POSTs it to ServiceNow.

**Steps:**

1. **Extract fields** — Parses all `### Section Name` headings from the issue body using a Python script.
2. **Load config** — Reads `.github/servicenow-config.yml` to get account, project, product, and other constants.
3. **Build payload** — Assembles a JSON object with issue fields + config constants + issue metadata (number, URL, title).
4. **POST to ServiceNow** — `curl -X POST` to `SERVICENOW_URL` with Basic Auth.
5. **Parse response** — Extracts `case_number` and `case_sys_id` from the JSON response.
6. **Post comments** — On success: posts a comment with the case number and a direct link to the case in ServiceNow. On failure: posts an error comment.

**Comments posted:**

- `✅ ServiceNow Case Created Successfully — [CS0001234](<link>)` — includes a clickable link to the case in ServiceNow
- `❌ ServiceNow Case Creation Failed — <error detail>`
- `👀 Watching for Change Requests` — posted alongside the success comment to indicate CR notifications are active

---

## Workflow: `servicenow-update-case.yml` (Reusable)

**Trigger:** `workflow_call` (called by `issue-servicenow.yml` on issue edit)

Sends a PATCH with the updated field values when an issue is edited after its case already exists.

**Condition for running:** The issue must have a `validation-passed` label, a "ServiceNow Case Created Successfully" comment must exist, and no Change Requests must have been created yet (editing is locked once a CR exists to prevent state drift).

**Comments posted:**

- `🔄 ServiceNow Case Updated — N field(s) updated: <field list>`
- `❌ ServiceNow Case Update Failed — <error detail>`

---

## Workflow: `sn-cr-notifier.yml` (CR Notifications)

**Trigger:** `repository_dispatch: [servicenow-cr-update]`

Receives events from ServiceNow Business Rules and posts Change Request updates as issue comments.

**Handles two actions:**

### `action: "created"` — New CR

1. Receives `github_issue_number`, `cr_number`, `cr_sys_id`, `cr_state`, `case_sys_id`.
2. Queries ServiceNow's Table API to fetch all CRs linked to `case_sys_id`.
3. Posts a comment listing the new CR and all existing CRs for the case.

**Comment format:**
```
New Change Request Created

CR Number: CHG0000001 (link)
State: New

All Change Requests for this Case (1)
- CHG0000001 - New

---
Automatically notified by ServiceNow
```

### `action: "state_changed"` — CR State Update

1. Receives `github_issue_number`, `cr_number`, `cr_state`, `previous_state`, optional `assigned_to`.
2. Resolves numeric state codes to display names using the state map.
3. Posts a state-change comment.

**Comment format:**
```
Change Request State Updated

CR Number: CHG0000001 (link)
State Change: New → Assess

---
Automatically notified by ServiceNow
```

**State map used by this workflow:**

| Code | Name |
|------|------|
| `-5` | New |
| `-4` | Assess |
| `-3` | Authorize |
| `-2` | Customer Approval |
| `-1` | Scheduled |
| `0` | Implement |
| `1` | Review |
| `2` | Customer Review |
| `3` | Rollback |
| `4` | Closed |
| `5` | Canceled |

---

## Workflow: `github-comment-to-sn.yml` (GitHub Comment → SN)

**Trigger:** `issue_comment: [created]`

Syncs human comments posted on a GitHub issue to the linked ServiceNow case's comments field.

**Steps:**

1. Skip if the comment author is a bot (GitHub Actions bot, etc.).
2. Find the "ServiceNow Case Created Successfully" comment on the issue.
3. Extract the `case_sys_id` from the URL query string in that comment.
4. PATCH the case's `comments` field with the new text, prepended with `@githubuser (GitHub Comment) <url>`.

This creates a one-way sync: every human comment on the GitHub issue appears as an Additional Comment on the ServiceNow case.

---

## Workflow: `sn-comment-to-github.yml` (SN Comment → GitHub)

**Trigger:** `repository_dispatch: [servicenow-note]`

Receives comment events from the optional ServiceNow Flow Designer flow and posts them as GitHub issue comments.

**Payload received:**

| Key | Value |
|-----|-------|
| `issue_number` | GitHub issue number |
| `note_text` | Comment body |
| `note_type` | `"additional_comments"` or `"work_notes"` |
| `case_number` | ServiceNow case number (e.g. `CS0001234`) |
| `sn_user` | ServiceNow user display name |

**Comment format:**
```
💬 ServiceNow Comment — Case CS0001234

<comment text>

Posted by Jane Smith via ServiceNow
```

---

## Workflow: `servicenow-inbound.yml` (Dispatch Handler)

**Trigger:** `repository_dispatch: [validation-passed]`

A thin dispatcher that invokes `servicenow-create-case.yml` when a `validation-passed` event is received via the repository_dispatch API. Used for external or manual dispatch-based testing scenarios.

---

## End-to-End Workflow Chain

```
Issue created/labeled
        ↓
issue-servicenow.yml
  ├── validates fields
  ├── on pass → calls servicenow-create-case.yml
  │                   └── POSTs to SN → posts success comment
  └── on edit (after case exists) → calls servicenow-update-case.yml
                                          └── PATCHes SN → posts update comment

ServiceNow creates CR
        ↓
Business Rule fires → sends repository_dispatch
        ↓
sn-cr-notifier.yml → posts CR comment on issue

User comments on issue
        ↓
github-comment-to-sn.yml → PATCHes SN case comments

ServiceNow agent posts case comment (optional)
        ↓
Flow Designer → sends repository_dispatch
        ↓
sn-comment-to-github.yml → posts comment on issue
```

---

## Bot Comments Reference

| Comment | Posted by | When |
|---------|-----------|------|
| `✅ Template Validation Passed` | `issue-servicenow.yml` | Field validation succeeds |
| `❌ Template Validation Failed` | `issue-servicenow.yml` | Field validation fails |
| `✅ ServiceNow Case Created Successfully` | `servicenow-create-case.yml` | Case created in SN |
| `❌ ServiceNow Case Creation Failed` | `servicenow-create-case.yml` | POST to SN fails |
| `👀 Watching for Change Requests` | `servicenow-create-case.yml` | Posted alongside case creation success |
| `🔄 ServiceNow Case Updated` | `servicenow-update-case.yml` | PATCH to SN succeeds |
| `❌ ServiceNow Case Update Failed` | `servicenow-update-case.yml` | PATCH to SN fails |
| `New Change Request Created` | `sn-cr-notifier.yml` | SN Business Rule fires on CR insert |
| `Change Request State Updated` | `sn-cr-notifier.yml` | SN Business Rule fires on CR state change |
| `💬 ServiceNow Comment` | `sn-comment-to-github.yml` | SN Flow Designer posts a case comment |
