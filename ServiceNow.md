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

    var issueNumber   = body.issue_number || '';
    var issueUrl      = body.issue_url || body.u_github_issue_url || '';

    // u_short_description = content of the "### Short Description" section in the GitHub issue body.
    // body.title = full GitHub issue title (e.g. "[SR-Change]: My title") — used only as fallback.
    var rawTitle      = (body.title || '').replace(/^\[SR-Change\]:\s*/i, '').trim();
    var title         = body.u_short_description || rawTitle || 'GitHub Issue';

    // HTML-formatted dump of all extracted issue fields + source footer
    var description       = body.description || '';
    var priority          = body.priority || '3 - Moderate';
    var catalogItem       = body.catalog_item || '';
    var caseType          = body.case_type || 'Service Request'; // u_case_type (reference)
    var category          = body.category || '';                 // integer/choice — resolved via setDisplayValue
    var account           = body.account || '';                  // reference — resolved via setDisplayValue
    var announcementType  = body.announcement_type || '';        // u_announcement_type (choice)
    var project           = body.project || '';                  // project (reference)
    var product           = body.product || '';                  // product (reference)
    var wso2Product       = body.wso2_product || '';             // u_wso2_product (reference)
    var environment       = body.u_project_environment || '';    // glide_list

    var affectedComp       = body.u_affected_component || '';       // string 4000
    var affectedSvc        = body.u_affected_services || '';        // string 4000
    var impact             = body.u_impact || '';                   // maps to standard impact integer
    var impactDescOverall  = body.u_impact_description_overall || '';  // string 4000
    var impactDescCustomer = body.u_impact_description_customer || ''; // string 4000
    var implPlan           = body.u_implementation_plan || '';     // string 4000
    var testPlan           = body.u_test_plan || '';               // string 4000
    var serviceOutage      = body.u_service_outage || '';          // string 4000
    var requestDetails     = body.u_request_details || '';         // string 1000

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

    // Standard impact integer: 1 = High, 2 = Medium, 3 = Low
    var impactMap = {
        'High': 1, '1 - High': 1, 'Critical': 1, '1 - Critical': 1,
        'Medium': 2, '2 - Medium': 2, 'Moderate': 2,
        'Low': 3, '3 - Low': 3, '4 - Low': 3
    };
    var snImpact = impact ? (impactMap[impact] || 2) : null;

    var gr = new GlideRecord('sn_customerservice_case');
    gr.initialize();

    // Standard task fields
    gr.short_description     = title;
    gr.description           = description;               // also stored in u_html_description below
    gr.priority              = snPriority;
    if (snImpact) gr.impact  = snImpact;                 // integer: 1=High 2=Medium 3=Low
    gr.u_github_issue_number = issueNumber;
    gr.u_github_issue_url    = issueUrl;

    // u_html_description (html, 5000 chars) — explicit HTML field for formatted display
    if (gr.isValidField('u_html_description')) gr.u_html_description = description;

    // --- Constants from .github/servicenow-config.yml ---

    // category: integer type with choice list — setDisplayValue resolves "Issue" → integer
    if (category) gr.category.setDisplayValue(category);

    // Reference fields: setDisplayValue looks up the record by display name
    if (account)      gr.account.setDisplayValue(account);
    if (caseType   && gr.isValidField('u_case_type'))    gr.u_case_type.setDisplayValue(caseType);
    if (project    && gr.isValidField('project'))        gr.project.setDisplayValue(project);
    if (product    && gr.isValidField('product'))        gr.product.setDisplayValue(product);
    if (wso2Product && gr.isValidField('u_wso2_product')) gr.u_wso2_product.setDisplayValue(wso2Product);

    // u_announcement_type: choice field — direct string assignment
    if (announcementType && gr.isValidField('u_announcement_type')) gr.u_announcement_type = announcementType;

    // --- Issue-body fields ---

    // u_project_environment: glide_list — pass comma-separated display values
    if (gr.isValidField('u_project_environment'))         gr.u_project_environment         = environment;
    if (gr.isValidField('u_catalog_item'))                gr.u_catalog_item                = catalogItem;
    if (gr.isValidField('u_affected_component'))          gr.u_affected_component          = affectedComp;
    if (gr.isValidField('u_affected_services'))           gr.u_affected_services           = affectedSvc;
    if (gr.isValidField('u_impact_description_overall'))  gr.u_impact_description_overall  = impactDescOverall;
    if (gr.isValidField('u_impact_description_customer')) gr.u_impact_description_customer = impactDescCustomer;
    if (gr.isValidField('u_implementation_plan'))         gr.u_implementation_plan         = implPlan;
    if (gr.isValidField('u_test_plan'))                   gr.u_test_plan                   = testPlan;
    if (gr.isValidField('u_service_outage'))              gr.u_service_outage              = serviceOutage;
    if (gr.isValidField('u_request_details'))             gr.u_request_details             = requestDetails;

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
