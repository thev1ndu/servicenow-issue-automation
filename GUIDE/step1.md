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

## 1.3 Configure `.github/servicenow-config.yml`

This file lives in the repository and holds the ServiceNow constants that are applied to **every** Case created by GitHub Actions. Edit it once and commit — no secrets or Actions variables needed.

**File location:** `.github/servicenow-config.yml`

```yaml
# ServiceNow constants applied to every Case created from a GitHub issue.
# Edit these values to match your ServiceNow instance — then commit.

case:
  category: "Issue"                                   # Category display value (e.g. "Issue", "Request")
  account: "Your Account Name"                        # Exact display name from the Account table in SN
  announcement_type: "General"                        # Announcement Type choice value from SN dropdown

project:
  name: "Your Project Name"                           # Project reference — exact display name from SN
  product: "Your Product Name"                        # Product reference — exact display name from SN
  wso2_product: "Your WSO2 Product Name"              # WSO2 Product reference — exact display name from SN
```

| Key | SN table / field | How to find the exact value |
|-----|------------------|-----------------------------|
| `case.category` | `sn_customerservice_case.category` | Open any Case → right-click **Category** → **Show value** |
| `case.account` | `sn_customerservice_case.account` (reference) | Open any Case → copy the **Account** display name exactly |
| `case.announcement_type` | `sn_customerservice_case.u_announcement_type` (choice) | Open any Case → right-click **Announcement Type** → **Show value** |
| `project.name` | `project` (reference) | Open any Case → copy the **Project** display name exactly |
| `project.product` | `product` (reference) | Open any Case → copy the **Product** display name exactly |
| `project.wso2_product` | `u_wso2_product` (reference) | Open any Case → copy the **WSO2 Product** display name exactly |

> **Important:** For reference fields (`account`, `case_type`, `project`, `product`, `wso2_product`), the value must **exactly** match the display name of the record in ServiceNow. If it doesn't match, ServiceNow silently leaves the field blank — no error is returned.

---

Next: [step2.md](step2.md) — Scripted REST API (inbound Case creation)
