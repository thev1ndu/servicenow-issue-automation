# ServiceNow Implementation

This document covers every component built inside ServiceNow to support the GitHub integration. Nothing here touches GitHub — all of it lives in your ServiceNow instance.

---

## What Was Built

| Component | Type | Purpose |
|-----------|------|---------|
| `u_github_issue_number` | Custom column on `sn_customerservice_case` | Stores GitHub issue number; used as idempotency key for inbound case creation |
| `u_github_issue_url` | Custom column on `sn_customerservice_case` | Stores full GitHub issue URL for traceability |
| `github.dispatch.config` | System property | Holds the GitHub PAT, org owner, and repo name used by outbound dispatch calls |
| `GitHub Case Integration` | Scripted REST API | Receives `POST /case` (create) and `PATCH /case` (update) calls from GitHub Actions |
| `GitHubRepositoryDispatch` | Script Include | Reusable HTTP client that sends `repository_dispatch` events to GitHub |
| `Notify GitHub on CR created` | Business Rule | Fires on CR insert; sends `servicenow-cr-update` event to GitHub |
| `Notify GitHub on CR state change` | Business Rule | Fires on CR state field change; sends `servicenow-cr-update` event to GitHub |

---

## Step 1 — Custom Columns on `sn_customerservice_case`

These two columns are the link between a ServiceNow Case and the GitHub issue that created it. Every inbound lookup and outbound notification depends on `u_github_issue_number`.

**Navigation:** System Definition → Tables → search `sn_customerservice_case` → Columns tab → New

| Setting | Column 1 | Column 2 |
|---------|----------|----------|
| Column label | GitHub Issue Number | GitHub Issue URL |
| Column name | `u_github_issue_number` | `u_github_issue_url` |
| Type | String (length 20) | URL |

Issue numbers are stored as strings to avoid integer-casting edge cases in GlideRecord queries.

---

## Step 2 — System Property `github.dispatch.config`

This property is read by the `GitHubRepositoryDispatch` Script Include every time ServiceNow sends an event to GitHub. It centralises credentials so they are not hardcoded in Business Rules.

**Navigation:** System Definition → System Properties → New

| Field | Value |
|-------|-------|
| Name | `github.dispatch.config` |
| Type | `String` (or `Password` to encrypt at rest) |
| Description | GitHub PAT + owner + repo for repository_dispatch calls |

**Value** (replace with real values):

```json
{
  "token": "github_pat_xxxxx",
  "owner": "your-github-org-or-user",
  "repo": "your-repo-name"
}
```

| Key | What to put |
|-----|-------------|
| `token` | GitHub Personal Access Token — fine-grained: **Contents: Read and write** on the repo; classic: `repo` scope |
| `owner` | GitHub organisation name or username (the segment before `/` in the repo URL) |
| `repo` | Repository name (the segment after `/` in the repo URL) |

The Script Include reads this property with `gs.getProperty('github.dispatch.config', '{}')` and parses it as JSON. If the JSON is malformed, a `gs.error` is logged and the dispatch is aborted.

---

## Step 3 — Scripted REST API: `GitHub Case Integration`

GitHub Actions sends all inbound case operations to this API. It exposes two resources under the base path `/api/<scope>/github_case/v1`.

**Navigation:** System Web Services → Scripted REST APIs → New

| Field | Value |
|-------|-------|
| Name | `GitHub Case Integration` |
| API ID | `github_case` |
| Description | Receives GitHub issue payloads and creates/updates Cases |

After saving, open the **Versions** tab and confirm the version is `v1`.

### 3.1 POST `/case` — Create Case

Receives the full issue payload from GitHub Actions and creates a Case on `sn_customerservice_case`.

**Resource settings:**

| Field | Value |
|-------|-------|
| Name | `Create Case` |
| HTTP Method | `POST` |
| Relative path | `/case` |

**Key behaviours in the script:**

- **Idempotency** — If a case with the same `u_github_issue_number` already exists the script returns the existing case (`200 OK`) without creating a duplicate.
- **Priority mapping** — `Critical → 1`, `High → 2`, `Moderate → 3`, `Low → 4`
- **Impact mapping** — `High → 1`, `Medium → 2`, `Low → 3`
- **Reference fields** — `account`, `project`, `product`, `u_wso2_product`, `u_case_type` are resolved by display name via `setDisplayValue`. If the display name does not exactly match a record in ServiceNow the field is silently left blank.
- **`category`** — An integer choice field resolved via `setDisplayValue("Issue")`.
- **HTML description** — The full formatted issue body is stored in both `description` and `u_html_description`.

**Full endpoint URL** (use this as `SERVICENOW_URL` in GitHub secrets):

```
https://<your-instance>.service-now.com/api/<scope>/github_case/v1/case
```

The `<scope>` segment is visible in the API record's **Base path** field.

**Payload field mapping:**

| GitHub payload key | ServiceNow field | Type | How set |
|-------------------|-----------------|------|---------|
| `u_short_description` | `short_description` | string 160 | direct |
| `title` | `short_description` (fallback) | string 160 | direct, `[SR-Change]:` stripped |
| `description` | `description` + `u_html_description` | string / html 5000 | direct |
| `priority` | `priority` | integer | mapped (Critical→1 … Low→4) |
| `u_impact` | `impact` | integer | mapped (High→1, Medium→2, Low→3) |
| `category` | `category` | integer choice | `setDisplayValue` |
| `account` | `account` | reference | `setDisplayValue` |
| `case_type` | `u_case_type` | reference | `setDisplayValue` |
| `announcement_type` | `u_announcement_type` | choice | direct |
| `project` | `project` | reference | `setDisplayValue` |
| `product` | `product` | reference | `setDisplayValue` |
| `wso2_product` | `u_wso2_product` | reference | `setDisplayValue` |
| `u_project_environment` | `u_project_environment` | glide_list | direct (comma-separated) |
| `u_affected_component` | `u_affected_component` | string 4000 | direct |
| `u_affected_services` | `u_affected_services` | string 4000 | direct |
| `u_service_outage` | `u_service_outage` | string 4000 | direct |
| `u_impact_description_overall` | `u_impact_description_overall` | string 4000 | direct |
| `u_impact_description_customer` | `u_impact_description_customer` | string 4000 | direct |
| `u_implementation_plan` | `u_implementation_plan` | string 4000 | direct |
| `u_test_plan` | `u_test_plan` | string 4000 | direct |
| `u_request_details` | `u_request_details` | string 1000 | direct |
| `issue_number` | `u_github_issue_number` | string 20 | direct — idempotency key |
| `issue_url` | `u_github_issue_url` | url 255 | direct |

> `u_monitoring_checks` and `u_maintenance_window` have no dedicated SN fields — their content is included in the HTML description dump.

**Success response (201):**

```json
{
  "case_number": "CS0001234",
  "sys_id": "abc123...",
  "case_sys_id": "abc123...",
  "message": "Case created successfully"
}
```

---

### 3.2 PATCH `/case` — Update Case

When a GitHub issue is edited after the Case already exists, Actions sends a PATCH with the updated field values.

**Resource settings:**

| Field | Value |
|-------|-------|
| Name | `Update Case` |
| HTTP Method | `PATCH` |
| Relative path | `/case` |

**Key behaviours:**

| Step | Detail |
|------|--------|
| Lookup | Finds the case by `u_github_issue_number` |
| Diff | Compares each incoming value to the current SN value |
| Update | Sets only fields that have changed |
| Work note | Adds a note: `Priority was updated from 3 - Moderate to 1 - Critical by johndoe` |
| Description refresh | Always updates `description` and `u_html_description` with the regenerated HTML |
| Response | Returns `case_number`, `sys_id`, `changes` (count), `change_log` (array) |

Fields tracked for work notes: Priority, Impact, Short Description, Impact Description (Overall), Impact Description (Customer), Environment, Affected Component, Affected Services, Service Outage, Implementation Plan, Test Plan, Request Details.

---

## Step 4 — Script Include: `GitHubRepositoryDispatch`

A single reusable class that wraps GitHub's `repository_dispatch` API. Both Business Rules instantiate this class to send events; they never make direct HTTP calls.

**Navigation:** System Definition → Script Includes → New

| Field | Value |
|-------|-------|
| Name | `GitHubRepositoryDispatch` |
| Client callable | false |
| Description | Sends repository_dispatch events to GitHub via RESTMessageV2 |

**How it works:**

1. `initialize()` calls `_loadConfig()` which reads `github.dispatch.config` via `gs.getProperty` and parses the JSON.
2. `send(eventType, clientPayload)` builds a `RESTMessageV2` POST to `https://api.github.com/repos/<owner>/<repo>/dispatches`.
3. Required headers: `Authorization: Bearer <token>`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28`.
4. GitHub returns `204 No Content` on success — this is checked to determine `ok: true`.
5. Any error is logged via `gs.error` and returned in the result object.

**Quick connectivity test** (run from System Definition → Scripts - Background):

```javascript
var d = new GitHubRepositoryDispatch();
var result = d.send('servicenow-cr-update', {
    github_issue_number: '1',
    action:              'state_changed',
    cr_number:           'CHG0000001',
    cr_sys_id:           gs.generateGUID(),
    cr_state:            'New',
    previous_state:      'Draft',
    case_sys_id:         gs.generateGUID()
});
gs.info(JSON.stringify(result));
```

Expected: `{"ok":true,"status":204,"body":""}`.

| Response | Cause |
|----------|-------|
| `401 / 403` | Token is invalid, expired, or missing the required scope |
| `404` | `owner` or `repo` is wrong in `github.dispatch.config` |
| `422` | `event_type` is missing or `client_payload` is not a JSON object |

---

## Step 5 — Business Rule: `Notify GitHub on CR created`

Fires immediately after a Change Request is inserted. Sends a `servicenow-cr-update` event with `action: "created"` to the linked GitHub issue.

**Navigation:** System Definition → Business Rules → New

| Field | Value |
|-------|-------|
| Name | `Notify GitHub on CR created` |
| Table | `Change Request [change_request]` |
| Active | checked |
| Advanced | checked |

**When to run tab:**

| Field | Value |
|-------|-------|
| When | `after` |
| Insert | checked |
| Update | unchecked |
| Delete | unchecked |

**What the script does:**

1. Reads `u_github_issue_number` directly from the CR.
2. Fallback: if the CR was created manually and the field is empty, looks up the parent Case and reads `u_github_issue_number` from there.
3. Aborts silently if no issue number is found (CR not linked to a GitHub issue).
4. Counts sibling CRs under the same parent Case (`total_crs`, `is_first_cr`).
5. Calls `GitHubRepositoryDispatch.send('servicenow-cr-update', { ... })`.

**Payload sent to GitHub:**

```json
{
  "github_issue_number": "42",
  "cr_number": "CHG0000001",
  "cr_sys_id": "<sys_id>",
  "cr_state": "New",
  "case_sys_id": "<parent_case_sys_id>",
  "action": "created",
  "total_crs": 1,
  "is_first_cr": true
}
```

---

## Step 6 — Business Rule: `Notify GitHub on CR state change`

Fires when the `state` field of a Change Request changes. Sends a `servicenow-cr-update` event with `action: "state_changed"` and includes the previous state.

**Navigation:** System Definition → Business Rules → New

| Field | Value |
|-------|-------|
| Name | `Notify GitHub on CR state change` |
| Table | `Change Request [change_request]` |
| Active | checked |
| Advanced | checked |

**When to run tab:**

| Field | Value |
|-------|-------|
| When | `after` |
| Update | checked |
| Filter Conditions | **State** `changes` |

**Payload sent to GitHub:**

```json
{
  "github_issue_number": "42",
  "cr_number": "CHG0000001",
  "cr_sys_id": "<sys_id>",
  "cr_state": "Assess",
  "case_sys_id": "<parent_case_sys_id>",
  "action": "state_changed",
  "previous_state": "New"
}
```

The `previous_state` value comes from the `previous` object available in Business Rule scripts. GitHub Actions maps both state values through its own state-name lookup table before posting the comment.

---

## CR State Number to Display Name Mapping

ServiceNow stores CR state as integers. GitHub Actions resolves them to human-readable names using this map (also maintained in `sn-cr-notifier.yml`):

| Number | Display name |
|--------|-------------|
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

The Business Rules send the display value directly (`current.state.getDisplayValue()`), but the workflow falls back to this map if a raw integer arrives.

---

## Optional — Flow Designer: SN Comment → GitHub

An optional Flow Designer flow can post ServiceNow case comments to the linked GitHub issue. This is documented separately in [guides/sn-comment-to-github.md](../guides/sn-comment-to-github.md).

- **Trigger:** `sn_customerservice_case` — Additional comments field changes
- **Action:** Reads the latest journal entry via `getJournalEntry(1)`, then calls `GitHubRepositoryDispatch.send('servicenow-note', { ... })`
- **GitHub side:** `sn-comment-to-github.yml` picks up the event and posts the comment on the issue

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Case not created, HTTP 400 | `issue_number` missing from payload | Check `servicenow-create-case.yml` build_payload step |
| Case not created, HTTP 404 | Wrong `SERVICENOW_URL` secret | Copy exact Base path from the Scripted REST API record |
| Reference fields blank in SN | Display name mismatch | Verify exact names in `.github/servicenow-config.yml` match SN records |
| CR comment not posted to GitHub | `github.dispatch.config` missing or wrong | Create/update the system property with correct token, owner, repo |
| `401` from GitHub dispatch | Token expired or wrong scope | Regenerate PAT with Contents: Read and write, update system property |
| `404` from GitHub dispatch | Wrong owner or repo in system property | Update `github.dispatch.config` |
| `gs.error` in system log | JSON in system property is malformed | Validate JSON before saving the property |
