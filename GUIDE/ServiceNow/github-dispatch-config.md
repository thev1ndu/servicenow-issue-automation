# Guide: github.dispatch.config System Property

`github.dispatch.config` is the single ServiceNow System Property that holds all GitHub
connection details used by every flow in this integration.

It is a JSON object **keyed by the exact Account display name** from the Customer Service Case.
Each key maps to the GitHub repo that team's cases should dispatch to. The script in each flow
reads the Account field from the case and uses it as the lookup key — no hardcoding anywhere.

---

## Format

```json
{
  "<Account display name>": {
    "token": "github_pat_...",
    "owner": "<org-or-user>",
    "repo":  "<repository-name>"
  }
}
```

| Field | Description |
|---|---|
| `<Account display name>` | Exact value of the **Account** field on the case — copy it character-for-character |
| `token` | GitHub Personal Access Token (classic) with `repo` scope |
| `owner` | GitHub org name or username that owns the repo |
| `repo` | GitHub repository name |

---

## Example — two teams

```json
{
  "Customer Portal Customer Account1": {
    "token": "github_pat_11A4TTP6I0pVPVtRXHzPEg_...",
    "owner": "thev1ndu",
    "repo":  "servicenow-issue-automation"
  },
  "Asgardeo Customer Account": {
    "token": "github_pat_...",
    "owner": "wso2",
    "repo":  "asgardeo-issues"
  }
}
```

A case with **Account = "Customer Portal Customer Account1"** dispatches to `servicenow-issue-automation`.
A case with **Account = "Asgardeo Customer Account"** dispatches to `asgardeo-issues`.
Both can run concurrently — each case carries its own routing key.

---

## How flows use this

Every Script Step 1 reads the Account field and passes it as `account_name`:

```javascript
var accountName = inputs.account_name + '';   // data pill: Account > Name
outputs.account_name = accountName;
```

Every Script Step 2 uses it as the lookup key:

```javascript
var config = JSON.parse(configJson)[inputs.account_name];
if (!config) {
  gs.error('No config entry for account "' + inputs.account_name + '" in github.dispatch.config');
  outputs.http_status = 'config_missing';
  outputs.success     = 'false';
  return;
}
var endpoint = 'https://api.github.com/repos/' + config.owner + '/' + config.repo + '/dispatches';
```

No constant, no hardcoding — the account name on the case drives everything.

---

## How to update the property in ServiceNow

1. Go to: `All > System Properties > System Properties`
2. Search for `github.dispatch.config`
3. Click the record
4. Edit the **Value** field — paste the full JSON object
5. Check **Ignore cache** so the updated value takes effect immediately
6. Click **Update**

> **Tip:** Use a JSON formatter before pasting to catch syntax errors.
> A malformed value causes all flows to log `config_missing` and stop sending.

---

## How to add a new account / team

1. Open `github.dispatch.config` as above
2. Add a new key — copy the account display name exactly from ServiceNow:

```json
{
  "Customer Portal Customer Account1": { ... },
  "New Team Account Name": {
    "token": "github_pat_...",
    "owner": "new-org",
    "repo":  "new-repo"
  }
}
```

3. No flow changes needed — the account name on the case automatically routes to the new entry

---

## Finding the exact account display name

The key must match the Account field value character-for-character (case-sensitive, spaces included).

To find it:
1. Open a case that belongs to the team
2. Right-click the **Account** field label → **Show value** (or hover to see the raw value)
3. Copy that string exactly as the JSON key

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `config_missing` in flow logs | Property missing or JSON is invalid | Verify the property exists and value parses as valid JSON |
| `No config entry for account "..."` | Account name on the case does not match any key | Copy the account name exactly — check spaces, capitalisation, special characters |
| `http_status: 401` | Token expired or missing `repo` scope | Regenerate the PAT and update `token` in the property |
| `http_status: 404` | `owner` or `repo` is wrong | Check against the GitHub URL: `github.com/{owner}/{repo}` |
| Changes not taking effect | Property is cached | Enable **Ignore cache** on the property record |
| Account field is empty on some cases | Case was created without selecting an account | Ensure agents always set the Account field when creating cases |
