# Scripted REST APIs — GitHub Integration

Two Scripted REST API resources handle all case operations triggered by GitHub issues.

**Base path:** `/api/wso2/gh_integration`  
**Authentication:** Basic Auth (`SERVICENOW_USERNAME` / `SERVICENOW_PASSWORD` secrets)  
**Content-Type:** `application/json`

---

## POST `/api/wso2/gh_integration/case` — Create Case

Called by the **ServiceNow · Create Case** workflow when a GitHub issue passes template validation.

### Request Body

| Field | Type | Source | Description |
|---|---|---|---|
| `issue_number` | string | GitHub | GitHub issue number — **required** |
| `issue_url` | string | GitHub | Full URL of the GitHub issue |
| `title` | string | GitHub | Full issue title (e.g. `[SR-Change]: My request`) |
| `description` | string | Workflow | HTML-formatted dump of all extracted fields + source footer |
| `catalog_item` | string | Template / Label | ServiceNow catalog item name |
| `case_type` | string | Template / Label | Case type string (e.g. `Service Request`, `Incident`) |
| `priority` | string | Issue body | Normalised priority (e.g. `3 - Moderate`) |
| `category` | string | `servicenow-config.yml` | Category display value (e.g. `Issue`) |
| `account` | string | `servicenow-config.yml` | Account display name |
| `announcement_type` | string | `servicenow-config.yml` | Announcement type choice value |
| `project` | string | `servicenow-config.yml` | Project reference display name |
| `product` | string | `servicenow-config.yml` | Product reference display name |
| `wso2_product` | string | `servicenow-config.yml` | WSO2 Product reference display name |
| `u_project_environment` | string | Issue body | Comma-separated environment list |
| `u_request_details` | string | Issue body | Brief request/incident summary |
| `u_short_description` | string | Issue body | Short description (Normal Change only) |
| `u_impact` | string | Issue body | Impact value (Normal Change only) |
| `u_impact_description_overall` | string | Issue body | Overall impact description (Normal Change only) |
| `u_impact_description_customer` | string | Issue body | Customer impact description (Normal Change only) |
| `u_affected_component` | string | Issue body | Affected component(s) (Normal Change only) |
| `u_affected_services` | string | Issue body | Affected service(s) (Normal Change only) |
| `u_service_outage` | string | Issue body | Service outage/downtime details (Normal Change only) |
| `u_implementation_plan` | string | Issue body | Implementation plan (Normal Change only) |
| `u_test_plan` | string | Issue body | Test plan (Normal Change only) |

### Field Population by Template

| SN Field | Bug Report / Incident | Service Request-Generic | Normal Change Generic |
|---|---|---|---|
| `short_description` | Issue title | Issue title | `u_short_description` field → fallback to title |
| `priority` | Severity dropdown | Priority dropdown | Priority dropdown |
| `u_request_details` | Issue Summary | Request Details | _(not sent)_ |
| `description` | Detailed Description | Description | Description |
| `u_project_environment` | Affected Environments | Environment Details | Environment Details |
| `u_impact` | _(not sent)_ | _(not sent)_ | Impact |
| `u_impact_description_overall` | _(not sent)_ | _(not sent)_ | Impact Description (Overall) |
| `u_impact_description_customer` | _(not sent)_ | _(not sent)_ | Impact Description (Customer) |
| `u_affected_component` | _(not sent)_ | _(not sent)_ | Affected Component |
| `u_affected_services` | _(not sent)_ | _(not sent)_ | Affected Services |
| `u_service_outage` | _(not sent)_ | _(not sent)_ | Service Outage/Downtime |
| `u_implementation_plan` | _(not sent)_ | _(not sent)_ | Implementation Plan |
| `u_test_plan` | _(not sent)_ | _(not sent)_ | Test Plan |
| `catalog_item` | `Incident Management` | `General Requests` | `Generic Requests` |
| `case_type` | `Incident` | `Service Request` | `Service Request` |

### Priority Normalisation

Issue form options with qualifiers (e.g. `Critical (System Down)`) are stripped to the base word before mapping:

| Issue form value | Sent to ServiceNow |
|---|---|
| `Critical (System Down)` | `1 - Critical` |
| `High (Major Impact)` | `2 - High` |
| `Moderate (Minor Impact)` | `3 - Moderate` |
| `Low (Cosmetic)` | `4 - Low` |
| `Critical` / `High` / `Moderate` / `Low` | Same mapping |

### Idempotency

If a case already exists for the given `issue_number`, the API returns the existing case with HTTP `200` — no duplicate is created.

### Response

**201 Created**
```json
{
  "case_number": "CS0012345",
  "sys_id": "abc123...32chars",
  "case_sys_id": "abc123...32chars",
  "message": "Case created successfully"
}
```

**200 OK** _(duplicate — case already exists)_
```json
{
  "case_number": "CS0012345",
  "sys_id": "abc123...32chars",
  "case_sys_id": "abc123...32chars",
  "message": "Case already exists for this issue"
}
```

**400 Bad Request**
```json
{ "error": "issue_number is required" }
```

**500 Internal Server Error**
```json
{ "error": "Case creation failed" }
```

---

## PATCH `/api/wso2/gh_integration/case` — Update Case

Called by the **ServiceNow · Update Case** workflow when a GitHub issue that already has a linked case is edited (before a CR is created).

### Request Body

| Field | Type | Description |
|---|---|---|
| `issue_number` | string | GitHub issue number — **required**, used to look up the existing case |
| `github_user` | string | GitHub login of the user who edited the issue |
| `title` | string | Updated issue title |
| `description` | string | HTML-formatted dump of all current field values |
| `priority` | string | Normalised priority string |
| `u_impact` | string | Impact value |
| `u_short_description` | string | Short description (Normal Change only) |
| `u_project_environment` | string | Comma-separated environment list |
| `u_request_details` | string | Brief request/incident summary |
| `u_impact_description_overall` | string | Overall impact description |
| `u_impact_description_customer` | string | Customer impact description |
| `u_affected_component` | string | Affected component(s) |
| `u_affected_services` | string | Affected service(s) |
| `u_service_outage` | string | Service outage/downtime details |
| `u_implementation_plan` | string | Implementation plan |
| `u_test_plan` | string | Test plan |

### Behaviour

- Only fields that have **changed** from their current ServiceNow value are written.
- A work note is added to the case listing each changed field with old → new values.
- `description` / `u_html_description` are always refreshed even if no tracked fields changed.
- Update is blocked (workflow does not call PATCH) once a Change Request has been created for the case.

### Response

**200 OK**
```json
{
  "case_number": "CS0012345",
  "sys_id": "abc123...32chars",
  "case_sys_id": "abc123...32chars",
  "changes": 2,
  "change_log": [
    { "field": "Priority", "old": "3 - Moderate", "new": "2 - High" },
    { "field": "Affected Component", "old": null, "new": "API Gateway" }
  ],
  "message": "Case updated with 2 change(s)"
}
```

**400 Bad Request**
```json
{ "error": "issue_number is required" }
```

**404 Not Found**
```json
{ "error": "No case found for issue 123" }
```

---

## Constants from `servicenow-config.yml`

The following fields are injected automatically by the workflow from `.github/servicenow-config.yml` and are the same for every case regardless of template:

| Field | Config key | SN field type |
|---|---|---|
| `category` | `case.category` | Integer/choice — resolved via `setDisplayValue` |
| `account` | `case.account` | Reference — resolved via `setDisplayValue` |
| `announcement_type` | `case.announcement_type` | Choice — direct string assignment |
| `project` | `project.name` | Reference — resolved via `setDisplayValue` |
| `product` | `project.product` | Reference — resolved via `setDisplayValue` |
| `wso2_product` | `project.wso2_product` | Reference — resolved via `setDisplayValue` |
