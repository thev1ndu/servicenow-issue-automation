# GitHub ↔ ServiceNow Integration

Raise a GitHub issue → a ServiceNow Case is created automatically. Change Requests, assignments, closures, and comments all sync back to GitHub. No one needs to touch ServiceNow manually.

---

## Templates

Three issue templates are available. Select the one that fits your request — blank issues are disabled.

| Template | Use it when | Auto-applied labels |
|---|---|---|
| **Bug Report / Incident** | Reporting a bug or system incident | `Type/Incident`, `bug` |
| **Service Request-Generic** | General service or information request | `Type/ServiceRequest` |
| **Normal Change Generic** | Requesting a planned change with full implementation details | `SRType/Normal Change`, `CatalogueItem/Generic Requests` |

Labels are applied automatically when you submit the issue — you do not need to add them manually.

---

## How to Raise a Request

### Bug Report / Incident

1. Go to **Issues → New issue** and select **Bug Report / Incident**
2. Fill in all four fields:
   - **Issue Summary** — one-line description of the problem
   - **Detailed Description** — steps to reproduce, expected vs actual behaviour, error messages
   - **Severity** — `Critical (System Down)` · `High (Major Impact)` · `Moderate (Minor Impact)` · `Low (Cosmetic)`
   - **Affected Environments** — select all that apply
3. Submit — automation starts immediately

---

### Service Request-Generic

1. Go to **Issues → New issue** and select **Service Request-Generic**
2. Fill in all four fields:
   - **Request Details** — brief summary of what you need
   - **Description** — full details of the request
   - **Priority** — `Critical` · `High` · `Moderate` · `Low`
   - **Environment Details** — select all that apply
3. Submit — automation starts immediately

---

### Normal Change Generic

1. Go to **Issues → New issue** and select **Service Request-Normal Change Generic**
2. Keep the title prefix exactly as pre-filled: `[SR-Change]: ` followed by your summary
   - Example: `[SR-Change]: Upgrade API Gateway to v3.2 in Production`
3. Fill in all 14 fields — none can be left blank
4. Submit — automation starts immediately

**Required fields:**

| Field | Notes |
|---|---|
| Short Description | One-line summary |
| Description | Full description of the change |
| Priority | `Critical` · `High` · `Moderate` · `Low` |
| Impact | `High` · `Medium` · `Low` |
| Impact Description (Overall) | System/application impact |
| Impact Description (Customer) | Customer/business impact |
| Environment Details | All affected environments |
| Affected Component | Component(s) being changed |
| Affected Services | Service(s) affected |
| Service Outage/Downtime | Expected downtime details |
| Is a maintenance window required or not | `Yes` · `No` · `TBD` |
| Implementation Plan | Step-by-step plan |
| Test Plan | How the change will be verified |
| Monitoring Checks | Staging checks passed? |

---

## What Happens Automatically

### 1 — Validation

The workflow checks that all required fields are filled and the template marker is present. For Normal Change, it also verifies the title starts with `[SR-Change]:`.

- **Pass** → `✅ Template Validation Passed` comment posted, `validation-passed` label added
- **Fail** → `❌ Template Validation Failed` comment posted explaining what is wrong

If validation fails, fix the issue body and the workflow re-runs automatically on your next edit.

### 2 — Case Creation

After validation, the workflow creates a ServiceNow Case and posts:

```
✅ ServiceNow Case Created — #CS0012345 [2026-05-14 10:30:00 IST]

Catalog: ...  ·  Type: ...
Priority: ...  ·  Environment: ...
Project: ...
Product: ...
```

Click the case number to open it directly in ServiceNow.

### 3 — Case Updates

If you edit the issue **before any Change Request has been created**, your changes are automatically sent to the ServiceNow case. A comment confirms how many fields were updated.

Once a Change Request exists, issue edits are no longer sent — post a comment instead (it syncs automatically).

### 4 — Change Request Notifications

When a ServiceNow agent creates or updates a Change Request, comments appear automatically:

**CR Created:**
```
💬 ServiceNow Change Request — Created #CHG0038549 [time]

State: New

All Change Requests for this Case (1)
- CHG0038549 - New
```

**CR State Changed:**
```
💬 ServiceNow Change Request — Updated #CHG0038549 [time]

New → Assess
Assigned To: Jane Smith
```

**Schedule Updated:**
```
💬 ServiceNow Change Request — Schedule Updated #CHG0038549 [time]

State: Assess
Assigned To: Jane Smith

⏰ Change Window: Thu, 15 May 2026 23:08:00 IST → Fri, 22 May 2026 23:08:03 IST
```

### 5 — Comment Sync

**You → ServiceNow:** Any comment you post on the GitHub issue is automatically copied to the ServiceNow case.

**ServiceNow → GitHub:** Comments posted by ServiceNow agents on the case appear on the GitHub issue.

### 6 — Case Closure & Assignment

```
✅ ServiceNow Case Closed — #CS0012345 [time]
👤 ServiceNow Case Assigned — #CS0012345 [time]
```

When the case is closed in ServiceNow, the GitHub issue is also closed automatically.

---

## Bot Comments Reference

| Comment | When |
|---|---|
| `✅ Template Validation Passed` | All fields valid, case creation triggered |
| `❌ Template Validation Failed` | Missing or invalid fields — read it for details |
| `✅ ServiceNow Case Created — #CSxxxxxxx` | Case created in ServiceNow |
| `❌ ServiceNow Case Creation Failed` | API call to ServiceNow failed |
| `🔄 ServiceNow Case Updated` | Issue edit applied to the case |
| `💬 ServiceNow Change Request — Created` | CR opened in ServiceNow |
| `💬 ServiceNow Change Request — Updated` | CR state changed |
| `💬 ServiceNow Change Request — Schedule Updated` | CR change window updated |
| `✅ ServiceNow Case Closed — #CSxxxxxxx` | Case closed, GitHub issue closed |
| `👤 ServiceNow Case Assigned — #CSxxxxxxx` | Case assigned to a team member |
| `💬 ServiceNow Comment` | ServiceNow agent posted on the case |

---

## Troubleshooting

**Validation failed — what do I do?**
Read the failure comment. It names the specific field or check that failed. Fix the issue body and save — the workflow re-runs automatically.

**The title validation fails on Normal Change.**
The title must start exactly with `[SR-Change]: ` (including the colon and space). The template prefills this — do not remove it.

**I edited the issue but no update comment appeared.**
A Change Request already exists. Once a CR is created, issue edits no longer sync to ServiceNow. Add a comment instead — it will be copied to the case.

**"ServiceNow Case Creation Failed" appeared.**
This is a configuration or credentials problem — do not close and reopen the issue. Contact the team maintaining this repository. The workflow will retry when configuration is corrected.

**Comments are not appearing in ServiceNow.**
The case must have been created first (look for a `✅ Case Created` comment). Comment sync requires a linked case. If case creation failed, comments will not sync.

**Do I need a ServiceNow account?**
No. All case and CR information is visible in the issue comments. Links in the comments open ServiceNow records directly if you do have access.

---

## Repository Structure

```
.github/
├── workflows/
│   ├── issue-servicenow.yml           Main orchestrator — validates and routes
│   ├── servicenow-create-case.yml     Reusable: POST case to ServiceNow
│   ├── servicenow-update-case.yml     Reusable: PATCH case on issue edit
│   ├── sn-cr-notifier.yml             Inbound: CR notifications → GitHub
│   ├── github-comment-to-sn.yml       Outbound: GitHub comments → ServiceNow
│   ├── sn-comment-to-github.yml       Inbound: ServiceNow comments → GitHub
│   ├── sn-case-updates.yml            Inbound: case closed / assigned → GitHub
│   └── sync-labels.yml                Syncs labels.yml to the repository
├── ISSUE_TEMPLATE/
│   ├── incident.yml                   Bug Report / Incident
│   ├── sr-generic.yml                 Service Request-Generic
│   └── sr-normal-change.yml           Normal Change Generic
├── labels.yml                         Label definitions
└── servicenow-config.yml              ServiceNow constants (account, project, product)

GUIDE/ServiceNow/
├── scripted-rest-apis.md              POST and PATCH API documentation
├── sn-cr-notifier.md                  CR notification setup in ServiceNow
├── sn-case-updates.md                 Case closure/assignment setup
└── sn-comment-to-github.md           Comment sync setup
```

---

## Secrets Required

Set these in **Repository Settings → Secrets and variables → Actions**:

| Secret | Value |
|---|---|
| `SERVICENOW_URL` | Full API endpoint: `https://<instance>.service-now.com/api/<scope>/gh_integration/case` |
| `SERVICENOW_UI_URL` | Base URL: `https://<instance>.service-now.com` |
| `SERVICENOW_USERNAME` | ServiceNow user with REST API access |
| `SERVICENOW_PASSWORD` | Password for the above user |
