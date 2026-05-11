## Step 4 — End-to-end testing and troubleshooting

### Test A: CR created → GitHub issue comment

1. Ensure `github.dispatch.config` is set correctly (Step 1).
2. Ensure the CR has a valid GitHub issue number field (example: `change_request.u_github_issue_number`).
3. Create a Change Request.
4. Confirm in GitHub:
   - Actions run: **ServiceNow inbound**
   - Issue comment: “New Change Request Created”

### Test B: CR state changed → GitHub issue comment

1. Update CR state.
2. Confirm in GitHub:
   - Actions run: **ServiceNow inbound**
   - Issue comment: “Change Request State Updated”

---

## Common errors

### ServiceNow returns HTTP 401 / 403 to GitHub API

- Token is invalid or lacks access to the repo.
- Fix: create a PAT with access to the repository and update `github.dispatch.config.token`.

### ServiceNow returns HTTP 404

- `owner` or `repo` wrong.
- Fix: update `github.dispatch.config.owner` / `repo`.

### GitHub Actions run happens, but no comment is posted

The inbound workflow requires:

- `client_payload.github_issue_number`
- `client_payload.cr_number`
- `client_payload.action` (`created` or `state_changed`)

Fix the Flow script payload.

### “All Change Requests for this Case” list is missing or only shows one item

GitHub queries ServiceNow using `client_payload.case_sys_id`.

- If `case_sys_id` is missing, GitHub cannot query all CRs.
- If ServiceNow credentials/secrets in GitHub are wrong, query will fail and GitHub falls back to listing only the current CR.

