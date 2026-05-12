# Step 2 — Scripted REST API: inbound Case creation endpoint

GitHub Actions POSTs to this endpoint to create a ServiceNow Case. You build it once; it handles all catalog item types.

---

## 2.1 Create the Scripted REST API

1. Go to **System Web Services → Scripted REST APIs**.
2. Click **New**.
3. Fill in:

| Field | Value |
|-------|-------|
| Name | `GitHub Case Integration` |
| API ID | `github_case` |
| Description | Receives GitHub issue payloads and creates Cases |

4. Leave **Enforce ACL** checked (default).
5. Click **Submit**.

---

## 2.2 Set the API version

1. Open the API you just created.
2. Click the **Versions** tab.
3. Click the default version (usually `v1`) to open it.
4. Confirm **Version** is `v1` and **Base path** contains `github_case`.

The full base URL will be:
```
https://<your-instance>.service-now.com/api/<scope>/github_case/v1
```

---

## 2.3 Add the POST /case resource

1. Inside the version, click the **Resources** tab.
2. Click **New**.
3. Fill in:

| Field | Value |
|-------|-------|
| Name | `Create Case` |
| HTTP Method | `POST` |
| Relative path | `/case` |

4. Paste the script below into the **Script** editor.
5. Click **Submit**.

### Resource script

```javascript
(function process(/*RESTAPIRequest*/ request, /*RESTAPIResponse*/ response) {

    var body = request.body.data;

    var issueNumber   = body.issue_number || '';
    var issueUrl      = body.issue_url || body.u_github_issue_url || '';

    // u_short_description = content of the "### Short Description" section in the GitHub issue body.
    // body.title = full GitHub issue title ("[SR-Change]: ...") — used only as fallback.
    var rawTitle      = (body.title || '').replace(/^\[SR-Change\]:\s*/i, '').trim();
    var title         = body.u_short_description || rawTitle || 'GitHub Issue';

    var description   = body.description || '';
    var priority      = body.priority || '3 - Moderate';
    var catalogItem   = body.catalog_item || 'General Requests';
    var caseType      = body.case_type || 'Service Request';
    var project       = body.project || '';
    var product       = body.product || '';
    var environment   = body.u_project_environment || '';
    var githubUser    = body.github_user || '';
    var labels        = body.labels || '';

    var affectedComp       = body.u_affected_component || '';
    var affectedSvc        = body.u_affected_services || '';
    var impact             = body.u_impact || '';
    var impactDescOverall  = body.u_impact_description_overall || '';
    var impactDescCustomer = body.u_impact_description_customer || '';
    var implPlan           = body.u_implementation_plan || '';
    var testPlan           = body.u_test_plan || '';
    var monChecks          = body.u_monitoring_checks || '';
    var maintWindow        = body.u_maintenance_window || '';
    var serviceOutage      = body.u_service_outage || '';
    var requestDetails     = body.u_request_details || '';

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

    gr.short_description     = title;
    gr.description           = description;
    gr.priority              = snPriority;
    if (snImpact) gr.impact  = snImpact;
    gr.u_github_issue_number = issueNumber;
    gr.u_github_issue_url    = issueUrl;

    if (gr.isValidField('u_catalog_item'))                gr.u_catalog_item                = catalogItem;
    if (gr.isValidField('u_case_type'))                   gr.u_case_type                   = caseType;
    if (gr.isValidField('u_project'))                     gr.u_project                     = project;
    if (gr.isValidField('u_product'))                     gr.u_product                     = product;
    if (gr.isValidField('u_project_environment'))         gr.u_project_environment         = environment;
    if (gr.isValidField('u_affected_component'))          gr.u_affected_component          = affectedComp;
    if (gr.isValidField('u_affected_services'))           gr.u_affected_services           = affectedSvc;
    if (gr.isValidField('u_impact'))                      gr.u_impact                      = impact;
    if (gr.isValidField('u_impact_description_overall'))  gr.u_impact_description_overall  = impactDescOverall;
    if (gr.isValidField('u_impact_description_customer')) gr.u_impact_description_customer = impactDescCustomer;
    if (gr.isValidField('u_implementation_plan'))         gr.u_implementation_plan         = implPlan;
    if (gr.isValidField('u_test_plan'))                   gr.u_test_plan                   = testPlan;
    if (gr.isValidField('u_monitoring_checks'))           gr.u_monitoring_checks           = monChecks;
    if (gr.isValidField('u_maintenance_window'))          gr.u_maintenance_window          = maintWindow;
    if (gr.isValidField('u_service_outage'))              gr.u_service_outage              = serviceOutage;
    if (gr.isValidField('u_request_details'))             gr.u_request_details             = requestDetails;
    if (gr.isValidField('u_github_user'))                 gr.u_github_user                 = githubUser;
    if (gr.isValidField('u_labels'))                      gr.u_labels                      = labels;

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

## 2.4 Note the endpoint URL

After saving, the full endpoint your GitHub secret `SERVICENOW_URL` must point to is:

```
https://<your-instance>.service-now.com/api/<scope>/github_case/v1/case
```

The `<scope>` part is shown in the API's **API ID** field as a namespaced path (e.g. `x_12345_github_case` or just `github_case`). Open the API record and copy the **Base path** value to confirm.

---

## 2.5 Payload field mapping reference

This table shows what the GitHub workflow sends and how each key maps to a Case field:

| Payload key | ServiceNow field | Notes |
|-------------|-----------------|-------|
| `u_short_description` | `short_description` | Content of `### Short Description` in issue body |
| `title` | `short_description` (fallback) | GitHub issue title stripped of `[SR-Change]:` prefix |
| `description` | `description` | HTML-formatted dump of all extracted issue fields |
| `priority` | `priority` | Mapped: Critical→1, High→2, Moderate→3, Low→4 |
| `u_impact` | `impact` (integer) + `u_impact` | Mapped: High→1, Medium→2, Low→3 |
| `u_impact_description_overall` | `u_impact_description_overall` | If field exists on table |
| `u_impact_description_customer` | `u_impact_description_customer` | If field exists on table |
| `u_project_environment` | `u_project_environment` | If field exists on table |
| `u_affected_component` | `u_affected_component` | If field exists on table |
| `u_affected_services` | `u_affected_services` | If field exists on table |
| `u_service_outage` | `u_service_outage` | If field exists on table |
| `u_maintenance_window` | `u_maintenance_window` | If field exists on table |
| `u_implementation_plan` | `u_implementation_plan` | If field exists on table |
| `u_test_plan` | `u_test_plan` | If field exists on table |
| `u_monitoring_checks` | `u_monitoring_checks` | If field exists on table |
| `issue_number` | `u_github_issue_number` | Used for idempotency check |
| `issue_url` | `u_github_issue_url` | GitHub issue URL |
| `project` | `u_project` | If field exists on table |
| `product` | `u_product` | If field exists on table |
| `github_user` | `u_github_user` | If field exists on table |
| `labels` | `u_labels` | If field exists on table |

Fields guarded by `isValidField()` are silently skipped if the column does not exist on the table — no error is thrown.

---

Next: [step3.md](step3.md) — Script Include + Business Rules (outbound CR notifications)
