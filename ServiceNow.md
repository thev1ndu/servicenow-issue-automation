# ServiceNow Configuration Reference

This document records exactly what was configured on the ServiceNow instance to support the GitHub ↔ ServiceNow integration. Use it as a reference when setting up a new instance or verifying an existing one.

---

## What was built in ServiceNow

| What | Where in ServiceNow | Name / ID |
|------|---------------------|-----------|
| Custom column | `sn_customerservice_case` table | `u_github_issue_number` (String, length 20) |
| Custom column | `sn_customerservice_case` table | `u_github_issue_url` (URL) |
| Scripted REST API | System Web Services → Scripted REST APIs | `GitHub Case Integration` |
| REST Resource (POST) | Inside the API above | `/case` |
| Script Include | System Definition → Script Includes | `GitHubRepositoryDispatch` |
| System Properties | System Definition → System Properties | `github.dispatch.config` |
| Business Rule | `change_request`, after insert | `Notify GitHub on CR created` |
| Business Rule | `change_request`, after update | `Notify GitHub on CR state change` |

---

## 1. Custom columns on `sn_customerservice_case`

Go to **System Definition → Tables**, find `sn_customerservice_case`, open the **Columns** tab, and add both:

| Setting | Column 1 | Column 2 |
|---------|----------|----------|
| Column label | GitHub Issue Number | GitHub Issue URL |
| Column name | `u_github_issue_number` | `u_github_issue_url` |
| Type | String (length 20) | URL |

---

## 2. Scripted REST API — inbound endpoint for GitHub Actions

GitHub Actions POSTs to this endpoint when creating a case (`SERVICENOW_URL` secret).

- **Name:** `GitHub Case Integration`
- **API ID:** `github_case`
- **Version:** `v1`
- **Base path:** `/api/<your-scope>/github_case`

**Resource:**
- **Name:** `Create Case`
- **HTTP Method:** `POST`
- **Relative path:** `/case`

**Full URL format:**
```
https://<your-instance>.service-now.com/api/<scope>/github_case/v1/case
```

**Resource script:**

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    var body = request.body.data;

    var issueNumber    = body.issue_number || '';
    var issueUrl       = body.issue_url || body.u_github_issue_url || '';
    var title          = body.title || body.u_short_description || 'GitHub Issue - No Title';
    var description    = body.description || '';
    var priority       = body.priority || '3 - Moderate';
    var catalogItem    = body.catalog_item || 'General Requests';
    var caseType       = body.case_type || 'Service Request';
    var project        = body.project || '';
    var product        = body.product || '';
    var environment    = body.u_project_environment || '';
    var githubUser     = body.github_user || '';
    var labels         = body.labels || '';

    var affectedComp   = body.u_affected_component || '';
    var affectedSvc    = body.u_affected_services || '';
    var impact         = body.u_impact || '';
    var implPlan       = body.u_implementation_plan || '';
    var testPlan       = body.u_test_plan || '';
    var monChecks      = body.u_monitoring_checks || '';
    var maintWindow    = body.u_maintenance_window || '';
    var serviceOutage  = body.u_service_outage || '';
    var requestDetails = body.u_request_details || '';

    if (!issueNumber) {
        response.setStatus(400);
        response.setBody({ error: 'issue_number is required' });
        return;
    }

    // Idempotency: return existing case if one already exists for this issue
    var existing = new GlideRecord('sn_customerservice_case');
    existing.addQuery('u_github_issue_number', issueNumber);
    existing.query();
    if (existing.next()) {
        response.setStatus(200);
        response.setBody({
            case_number: existing.number.toString(),
            sys_id: existing.sys_id.toString(),
            case_sys_id: existing.sys_id.toString(),
            message: 'Case already exists for this issue'
        });
        return;
    }

    var priorityMap = {
        '1 - Critical': 1, 'Critical': 1,
        '2 - High': 2,     'High': 2,
        '3 - Moderate': 3, 'Moderate': 3, 'Medium': 3,
        '4 - Low': 4,      'Low': 4
    };
    var snPriority = priorityMap[priority] || 3;

    var gr = new GlideRecord('sn_customerservice_case');
    gr.initialize();

    gr.short_description     = title;
    gr.description           = description;
    gr.priority              = snPriority;
    gr.u_github_issue_number = issueNumber;
    gr.u_github_issue_url    = issueUrl;

    if (gr.isValidField('u_catalog_item'))        gr.u_catalog_item        = catalogItem;
    if (gr.isValidField('u_case_type'))           gr.u_case_type           = caseType;
    if (gr.isValidField('u_project'))             gr.u_project             = project;
    if (gr.isValidField('u_product'))             gr.u_product             = product;
    if (gr.isValidField('u_project_environment')) gr.u_project_environment = environment;
    if (gr.isValidField('u_affected_component'))  gr.u_affected_component  = affectedComp;
    if (gr.isValidField('u_affected_services'))   gr.u_affected_services   = affectedSvc;
    if (gr.isValidField('u_impact'))              gr.u_impact              = impact;
    if (gr.isValidField('u_implementation_plan')) gr.u_implementation_plan = implPlan;
    if (gr.isValidField('u_test_plan'))           gr.u_test_plan           = testPlan;
    if (gr.isValidField('u_monitoring_checks'))   gr.u_monitoring_checks   = monChecks;
    if (gr.isValidField('u_maintenance_window'))  gr.u_maintenance_window  = maintWindow;
    if (gr.isValidField('u_service_outage'))      gr.u_service_outage      = serviceOutage;
    if (gr.isValidField('u_request_details'))     gr.u_request_details     = requestDetails;
    if (gr.isValidField('u_github_user'))         gr.u_github_user         = githubUser;
    if (gr.isValidField('u_labels'))              gr.u_labels              = labels;

    var sysId = gr.insert();

    if (!sysId) {
        gs.error('GitHubCaseIntegration: GlideRecord insert failed for issue ' + issueNumber);
        response.setStatus(500);
        response.setBody({ error: 'Case creation failed' });
        return;
    }

    gs.info('GitHubCaseIntegration: Created case ' + gr.number + ' for GitHub issue #' + issueNumber);

    response.setStatus(201);
    response.setBody({
        case_number:  gr.number.toString(),
        sys_id:       sysId.toString(),
        case_sys_id:  sysId.toString(),
        message:      'Case created successfully'
    });

})(request, response);
```

---

## 3. Script Include — `GitHubRepositoryDispatch`

Used by Business Rules and Flows to send outbound calls to GitHub's `repository_dispatch` API.

Configuration is read from the system property `github.dispatch.config` (JSON string):

```json
{
  "token": "github_pat_xxxxx",
  "owner": "your-github-org-or-user",
  "repo": "your-repo-name"
}
```

See [GUIDE/step1.md](GUIDE/step1.md) for the full Script Include source and setup steps.

---

## 4. Business Rule — `Notify GitHub on CR created`

Fires after a Change Request is inserted, if the parent case has a GitHub issue number.

- **Table:** `change_request`
- **When:** `after`
- **Insert:** ✅

```javascript
(function executeRule(current, previous) {

    if (!current.parent || !current.parent.isValidRecord()) return;

    var parentCase = new GlideRecord('sn_customerservice_case');
    if (!parentCase.get(current.parent.sys_id.toString())) return;

    var issueNumber = parentCase.u_github_issue_number.toString();
    if (!issueNumber) return;

    var sibling = new GlideRecord('change_request');
    sibling.addQuery('parent', parentCase.sys_id.toString());
    sibling.query();
    var total = sibling.getRowCount();

    var dispatcher = new GitHubRepositoryDispatch();
    dispatcher.send('servicenow-cr-update', {
        github_issue_number: issueNumber,
        cr_number:           current.number.toString(),
        cr_sys_id:           current.sys_id.toString(),
        cr_state:            current.state.getDisplayValue(),
        case_sys_id:         parentCase.sys_id.toString(),
        action:              'created',
        total_crs:           total,
        is_first_cr:         (total === 1)
    });

})(current, previous);
```

---

## 5. Business Rule — `Notify GitHub on CR state change`

Fires after a Change Request state changes.

- **Table:** `change_request`
- **When:** `after`
- **Update:** ✅
- **Condition:** `current.state.changes()`

```javascript
(function executeRule(current, previous) {

    if (!current.parent || !current.parent.isValidRecord()) return;

    var parentCase = new GlideRecord('sn_customerservice_case');
    if (!parentCase.get(current.parent.sys_id.toString())) return;

    var issueNumber = parentCase.u_github_issue_number.toString();
    if (!issueNumber) return;

    var dispatcher = new GitHubRepositoryDispatch();
    dispatcher.send('servicenow-cr-update', {
        github_issue_number: issueNumber,
        cr_number:           current.number.toString(),
        cr_sys_id:           current.sys_id.toString(),
        cr_state:            current.state.getDisplayValue(),
        cr_environment:      current.u_environment ? current.u_environment.getDisplayValue() : '',
        case_sys_id:         parentCase.sys_id.toString(),
        action:              'state_changed',
        previous_state:      previous.state.getDisplayValue()
    });

})(current, previous);
```

---

## Key design decisions

**Why a Scripted REST API for inbound instead of the OOB Case API?**
The OOB `POST /sn_customerservice/case` endpoint doesn't let you set custom `u_` fields in one shot cleanly, and doesn't return `case_number`/`case_sys_id` with the exact keys the GitHub workflow expects.

**Why the idempotency check on `u_github_issue_number`?**
The GitHub workflow can be re-triggered (on issue edits, label changes). The check prevents duplicate cases being created for the same issue.

**Why store `u_github_issue_number` as String, not Integer?**
ServiceNow GlideRecord string queries are simpler, and GitHub issue numbers are referenced as strings throughout the workflow.

**Why Business Rules instead of Flow Designer for CR notifications?**
Business Rules fire synchronously and reliably on insert/update with previous value access (`previous.state`). Flow Designer is also valid — see [GUIDE/step2.md](GUIDE/step2.md) and [GUIDE/step3.md](GUIDE/step3.md) for the Flow Designer equivalent.
