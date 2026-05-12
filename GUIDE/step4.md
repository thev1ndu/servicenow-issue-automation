# Step 4 — End-to-end testing and troubleshooting

## Test A: GitHub issue → ServiceNow Case (inbound)

1. In your GitHub repo, open a new issue using one of the SR templates.
2. Fill in all required fields. Do **not** leave `_No response_` in any section.
3. Set the issue title to start with `[SR-Change]:`.
4. Add both required labels: one `SRType/` and one `CatalogueItem/` label.
5. GitHub Actions validates the issue and posts "✅ Template Validation Passed".
6. The `servicenow-create-case` workflow runs and POSTs the payload to your Scripted REST API.
7. Confirm in GitHub: a bot comment "✅ ServiceNow Case Created Successfully" with the case number.
8. Confirm in ServiceNow: open the case and verify:
   - **Short description** = content of the `### Short Description` section from the issue body
   - **Description** = HTML-formatted summary of all extracted fields
   - **Priority** set correctly
   - **Impact** set correctly
   - `u_github_issue_number` = the GitHub issue number
   - `u_github_issue_url` = the GitHub issue URL

---

## Test B: ServiceNow CR created → GitHub comment (outbound)

1. In ServiceNow, open the Case created in Test A.
2. Create a Change Request linked to that Case.
   - Ensure the Business Rule can find the issue number (either via `u_github_issue_number` on the CR, or via the parent Case).
3. In GitHub → **Actions**, confirm the **ServiceNow inbound** workflow runs.
4. On the GitHub issue, confirm a bot comment with "New Change Request Created" and the CR number.

---

## Test C: ServiceNow CR state changed → GitHub comment (outbound)

1. Open the Change Request from Test B.
2. Change its **State** (e.g. New → Assess, or Assess → Authorize).
3. In GitHub → **Actions**, confirm the **ServiceNow inbound** workflow runs again.
4. On the GitHub issue, confirm a bot comment with "Change Request State Updated" and the state transition.

---

## Testing the inbound endpoint directly (without GitHub Actions)

Use this curl to verify your Scripted REST API accepts payloads and creates cases:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u "your-sn-user:your-sn-password" \
  -d '{
    "issue_number": "999",
    "issue_url": "https://github.com/org/repo/issues/999",
    "u_short_description": "Test case from curl",
    "title": "[SR-Change]: Test curl issue",
    "description": "<strong>SHORT DESCRIPTION:</strong><br/>Test case from curl<br/><br/>",
    "priority": "3 - Moderate",
    "u_impact": "Medium",
    "catalog_item": "Generic Requests",
    "case_type": "Service Request",
    "project": "Test Project",
    "product": "Test Product"
  }' \
  "https://<your-instance>.service-now.com/api/<scope>/github_case/v1/case"
```

Expected response (HTTP 201):
```json
{
  "case_number": "CS0000001",
  "sys_id": "...",
  "case_sys_id": "...",
  "message": "Case created successfully"
}
```

If you send the same `issue_number` a second time, you get HTTP 200 with `"message": "Case already exists for this issue"` (idempotency).

---

## Testing the outbound Script Include directly

Run from **System Definition → Scripts - Background**:

```javascript
var d = new GitHubRepositoryDispatch();
var result = d.send('servicenow-cr-update', {
    github_issue_number: '1',
    action:              'created',
    cr_number:           'CHG0000001',
    cr_sys_id:           'test-sys-id',
    cr_state:            'New',
    case_sys_id:         'test-case-sys-id'
});
gs.info('ok=' + result.ok + ' status=' + result.status);
```

Expected: `ok=true status=204`.

---

## Common errors and fixes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Case created but **Short description is empty** | Production SN script still has old version (reads `body.title` not `body.u_short_description`) | Replace the Scripted REST API `/case` script with the version in [step2.md](step2.md) |
| Case created but **Description is empty** | `body.description` not reaching `gr.description` | Verify the `/case` script sets `gr.description = description` where `description = body.description` |
| Extra fields appear in **Json Data** field | A custom field or Business Rule on the Case table is capturing the raw body | Check for Business Rules on `sn_customerservice_case` that write to `u_json_data`; remove or disable them |
| HTTP 400 — `issue_number is required` | The workflow payload is missing `issue_number` | Check the `build_payload` step in `servicenow-create-case.yml` |
| HTTP 401 / 403 from Scripted REST API | Wrong SN credentials in GitHub secrets | Update `SERVICENOW_USERNAME` / `SERVICENOW_PASSWORD` |
| HTTP 404 from Scripted REST API | `SERVICENOW_URL` secret has wrong path | Copy the exact URL from the API's Version record (Base path) |
| HTTP 404 from GitHub dispatch | Wrong `owner` or `repo` in `github.dispatch.config` | Fix the JSON in the system property |
| HTTP 401 / 403 from GitHub dispatch | PAT missing **Contents: Read and write** scope | Re-generate the PAT and update `github.dispatch.config.token` |
| GitHub Actions runs but no issue comment | `github_issue_number` wrong or missing in dispatch payload | Check Business Rule script — `issueNumber` must be the numeric GitHub issue number (as a string) |
| "All CRs" list shows only one entry | `case_sys_id` missing from dispatch payload | Add `case_sys_id: current.parent.toString()` to the Business Rule payload |
| `github.dispatch.config` parse error in SN logs | JSON malformed (trailing comma, unquoted keys) | Validate the JSON at jsonlint.com before saving the property |
