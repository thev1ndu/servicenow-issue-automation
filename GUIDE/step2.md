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

## 2.4 Note the endpoint URL

After saving, the full endpoint your GitHub secret `SERVICENOW_URL` must point to is:

```
https://<your-instance>.service-now.com/api/<scope>/github_case/v1/case
```

The `<scope>` part is shown in the API's **API ID** field as a namespaced path (e.g. `x_12345_github_case` or just `github_case`). Open the API record and copy the **Base path** value to confirm.

---

## 2.5 Payload field mapping reference

This table shows what the GitHub workflow sends and how each key maps to a Case field:

| Payload key | SN field | SN type | How set | Source |
|-------------|----------|---------|---------|--------|
| `u_short_description` | `short_description` | string 160 | direct | `### Short Description` in issue body |
| `title` | `short_description` (fallback) | string 160 | direct | GitHub issue title, `[SR-Change]:` stripped |
| `description` | `description` + `u_html_description` | string / html 5000 | direct | HTML dump of all issue fields + source footer |
| `priority` | `priority` | integer | mapped | Critical→1, High→2, Moderate→3, Low→4 |
| `u_impact` | `impact` | integer | mapped | High→1, Medium→2, Low→3 |
| `category` | `category` | **integer** | `setDisplayValue` | **`servicenow-config.yml`** → `case.category` |
| `account` | `account` | **reference** | `setDisplayValue` | **`servicenow-config.yml`** → `case.account` |
| `case_type` | `u_case_type` | **reference** | `setDisplayValue` | **`servicenow-config.yml`** → `case.case_type` |
| `announcement_type` | `u_announcement_type` | choice | direct | **`servicenow-config.yml`** → `case.announcement_type` |
| `project` | `project` | **reference** | `setDisplayValue` | **`servicenow-config.yml`** → `project.name` |
| `product` | `product` | **reference** | `setDisplayValue` | **`servicenow-config.yml`** → `project.product` |
| `wso2_product` | `u_wso2_product` | **reference** | `setDisplayValue` | **`servicenow-config.yml`** → `project.wso2_product` |
| `u_impact_description_overall` | `u_impact_description_overall` | string 4000 | direct | `### Impact Description (Overall)` |
| `u_impact_description_customer` | `u_impact_description_customer` | string 4000 | direct | `### Impact Description (Customer)` |
| `u_project_environment` | `u_project_environment` | glide_list | direct | `### Environment Details` (comma-separated) |
| `u_affected_component` | `u_affected_component` | string 4000 | direct | `### Affected Component` |
| `u_affected_services` | `u_affected_services` | string 4000 | direct | `### Affected Services` |
| `u_service_outage` | `u_service_outage` | string 4000 | direct | `### Service Outage/Downtime` |
| `u_implementation_plan` | `u_implementation_plan` | string 4000 | direct | `### Implementation Plan` |
| `u_test_plan` | `u_test_plan` | string 4000 | direct | `### Test Plan` |
| `u_request_details` | `u_request_details` | string 1000 | direct | `### Request Details` |
| `issue_number` | `u_github_issue_number` | string 20 | direct | GitHub issue number (idempotency key) |
| `issue_url` | `u_github_issue_url` | url 255 | direct | GitHub issue URL |

> **`setDisplayValue` note:** For reference fields, ServiceNow looks up the referenced record by its display name at insert time. If the name doesn't exactly match a record in the target table, the field will be left blank — no error is thrown. Verify the display name in `servicenow-config.yml` matches exactly what appears in ServiceNow.

> **`u_monitoring_checks` and `u_maintenance_window`:** These fields do not exist on `sn_customerservice_case`. Their content is included in the HTML description instead.
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
