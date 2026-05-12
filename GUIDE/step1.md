# Step 1 — Prerequisites: custom columns and system property

## 1.1 Add custom columns to `sn_customerservice_case`

These two columns link a ServiceNow Case back to the GitHub issue that created it.

1. Go to **System Definition → Tables**.
2. Search for `sn_customerservice_case` and open it.
3. Click the **Columns** tab.
4. Add each column below by clicking **New**:

| Setting | Column 1 | Column 2 |
|---------|----------|----------|
| Column label | GitHub Issue Number | GitHub Issue URL |
| Column name | `u_github_issue_number` | `u_github_issue_url` |
| Type | String (length 20) | URL |

5. Save both columns.

> **Why String for the issue number?**  
> GitHub issue numbers are queried and compared as strings throughout the integration, so String avoids integer-casting edge cases.

---

## 1.2 Create the system property `github.dispatch.config`

ServiceNow uses this property to authenticate and target the GitHub repo when sending outbound events.

1. Go to **System Definition → System Properties** (or search "System Properties" in the nav filter).
2. Click **New**.
3. Fill in:

| Field | Value |
|-------|-------|
| Name | `github.dispatch.config` |
| Type | `String` (or `Password` if your org encrypts integration credentials) |
| Description | GitHub PAT + owner + repo for repository_dispatch calls |

4. Set the **Value** to this JSON (replace with real values):

```json
{
  "token": "github_pat_xxxxx",
  "owner": "your-github-org-or-user",
  "repo": "your-repo-name"
}
```

| Key | What to put |
|-----|-------------|
| `token` | A GitHub Personal Access Token with **Contents: Read and write** permission on the target repo |
| `owner` | GitHub organisation name or username (the part before `/` in the repo URL) |
| `repo` | Repository name (the part after `/` in the repo URL) |

5. Save the property.

> **Token permissions required:**  
> Fine-grained token: `Contents: Read and write` on the specific repo.  
> Classic token: `repo` scope.

---

Next: [step2.md](step2.md) — Scripted REST API (inbound Case creation)
