# User Guide

This guide explains how the GitHub ↔ ServiceNow integration works from a user's perspective — how to raise a Service Request, what happens automatically, and how to track progress.

---

## What This System Does

When you create a GitHub issue using one of the SR templates, a ServiceNow Case is automatically created. From that point on, updates flow in both directions:

- GitHub issue edits update the ServiceNow case.
- ServiceNow Change Requests appear as comments on your GitHub issue.
- Comments you post on the GitHub issue are copied to the ServiceNow case.
- Comments posted by ServiceNow agents (if the optional flow is configured) appear on your GitHub issue.

You do not need to manually update ServiceNow or copy information between systems.

---

## How to Raise a Service Request

### 1 — Create a new issue

Go to the **Issues** tab of the repository and click **New issue**. You will see four SR templates — do not use the blank issue option (it is disabled).

| Template | Use it when |
|----------|------------|
| SR Generic Requests | Production incidents, complex changes, anything requiring implementation and test plans |
| SR Request Logs | You need logs extracted from a system — priority must be Critical |
| SR Standard Generic | Routine requests that do not need a full implementation plan |
| SR Information | You need information only; no change will be made |

Select the template that matches your request.

---

### 2 — Fill in all required fields

The template contains sections marked with `### Field Name`. Fill in every section — do not leave the default `_No response_` placeholder in any required field.

**Required fields per template:**

| Template | Required sections |
|----------|-----------------|
| SR Generic Requests | Short Description, Description, Priority, Impact, Impact Description (Overall), Impact Description (Customer), Environment Details, Affected Component, Affected Services, Service Outage/Downtime, Is a maintenance window required, Implementation Plan, Test Plan, Monitoring Checks |
| SR Request Logs | Short Description, Description, Priority (must be Critical), Impact, Impact Description, Customer Project, Environment Details |
| SR Standard Generic | Short Description, Description, Priority, Impact, Environment Details |
| SR Information | Request Description, Impact, Customer Project |

**Priority values accepted:** `Critical`, `High`, `Moderate`, `Low`

**Impact values accepted:** `High`, `Medium`, `Low`

---

### 3 — Set the issue title

The title must follow this format:

```
[SR-Change]: <short summary of your request>
```

Example: `[SR-Change]: Production API Gateway returning 502 errors`

The validation will fail if the title does not start with `[SR-Change]:`.

---

### 4 — Add the required labels

Before submitting (or immediately after), add two labels:

- **One SRType label** — `SRType/Normal Change`, `SRType/Standard Change`, or `SRType/Emergency Change`
- **One CatalogueItem label** — must match the template you used (e.g. `CatalogueItem/Generic Requests`)

The automation only runs after both labels are present.

---

### 5 — Submit and wait

Once both labels are added, the automation starts within seconds. Watch the issue comments — they report every step.

---

## What Happens Automatically

### Validation

The workflow checks your issue for:
- A recognised SR template marker in the body
- Both required labels (`SRType/*` and `CatalogueItem/*`)
- A title starting with `[SR-Change]:`
- All required fields filled in (no `_No response_` left)
- Template-specific rules (e.g. Priority = Critical for SR Request Logs)

**If validation fails:** A comment explains exactly what is wrong. The `SRType/*` and `CatalogueItem/*` labels are removed. Fix the issue and re-add the labels to trigger validation again.

**If validation passes:** A `✅ Template Validation Passed` comment is posted and the `validation-passed` label is added.

---

### Case Creation

After successful validation, the workflow:

1. Extracts all field values from the issue body.
2. Applies the constants from the repository config (account, project, product).
3. Creates a Case in ServiceNow via the Scripted REST API.
4. Posts a success comment with the case number and a direct link to the case.

**Success comment example:**
```
✅ ServiceNow Case Created Successfully

Case: CS0001234
https://<instance>.service-now.com/...

👀 Watching for Change Requests
```

From this point, a ServiceNow agent handles the case and will create Change Requests as needed.

---

### Case Updates (when you edit the issue)

If you edit the issue body **before any Change Request has been created**, the workflow automatically sends the updated field values to ServiceNow. A work note is added to the case listing every field that changed.

Once a Change Request exists, the case is locked against issue-edit updates to prevent state drift.

---

### Change Request Notifications

When a ServiceNow agent creates a Change Request linked to your case, a comment is automatically posted on your GitHub issue:

```
New Change Request Created

CR Number: CHG0000001 (link to ServiceNow)
State: New

All Change Requests for this Case (1)
- CHG0000001 - New

---
Automatically notified by ServiceNow
```

When the CR's state changes (e.g. New → Assess → Scheduled → Implement → Closed), another comment is posted:

```
Change Request State Updated

CR Number: CHG0000001 (link)
State Change: New → Assess

---
Automatically notified by ServiceNow
```

You do not need to log in to ServiceNow to track CR progress — all state transitions appear on the GitHub issue.

---

### Comment Sync

**GitHub → ServiceNow:** Any comment you post on the GitHub issue (except bot comments) is automatically copied to the ServiceNow case as an Additional Comment. The comment includes your GitHub username and a link back to the original comment.

**ServiceNow → GitHub (optional):** If the optional Flow Designer flow is configured, comments posted by ServiceNow agents on the case appear on the GitHub issue attributed to the agent's name.

---

## Tracking Your Request

Everything you need to know about your SR is visible on the GitHub issue:

| What to look for | Meaning |
|-----------------|---------|
| `validation-passed` label | Issue validated, case creation triggered |
| `✅ ServiceNow Case Created Successfully` comment | Case exists in ServiceNow — click the link to open it |
| `New Change Request Created` comment | ServiceNow has started work on a CR |
| `Change Request State Updated` comment | CR is progressing — the state change is shown |
| `🔄 ServiceNow Case Updated` comment | Your issue edit was applied to the case |

---

## Common Questions

**My validation failed — what do I do?**

Read the failure comment. It names the specific field or rule that failed. Fix the issue body and re-add the `SRType/*` and `CatalogueItem/*` labels to trigger validation again.

**Can I edit the issue after the case is created?**

Yes, but only before a Change Request is created. After a CR exists, edits to the issue are not sent to ServiceNow (the workflow skips the update). You can still add comments, which are synced to the case.

**Can I submit without labels?**

The automation will not run without both a `SRType/*` and a `CatalogueItem/*` label. Add both labels after filling the form — validation starts automatically.

**I see a comment saying "ServiceNow Case Creation Failed" — what now?**

This is typically a configuration problem (wrong credentials or endpoint URL). Contact the team maintaining the repository. Do not retry by closing and reopening the issue — the original issue will be retried when the configuration is fixed.

**Will I be notified when the CR closes?**

Yes. Every CR state change posts a comment on your GitHub issue, including the final `Closed` or `Canceled` transition.

**Do I need a ServiceNow account?**

No. Everything is visible on the GitHub issue. If you need to view the case or CR in detail, the comments include direct links that open the records in ServiceNow (you will need ServiceNow credentials to log in there).

---

## Full Flow Summary

```
You: Create issue from SR template
You: Fill all required fields
You: Set title to [SR-Change]: ...
You: Add SRType/* and CatalogueItem/* labels
         ↓
Automation: Validates the issue
         ↓
Automation: Creates ServiceNow Case
Automation: Posts case link on the issue
         ↓
ServiceNow agent: Reviews the case
ServiceNow agent: Creates a Change Request
         ↓
Automation: Posts "New Change Request Created" on the issue
         ↓
ServiceNow agent: Works the CR, changes its state
         ↓
Automation: Posts each state change on the issue
         ↓
ServiceNow agent: Closes the CR and Case
         ↓
Automation: Posts final state change on the issue
```
