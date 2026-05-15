# Guide: github.dispatch.config System Property

`github.dispatch.config` is the single ServiceNow System Property that holds all GitHub
connection details used by every flow in this integration.

It is a JSON object **keyed by repo name**. Each key maps to the credentials for that repo.
All flows read it at runtime — nothing is hardcoded in the REST Message record itself.

---

## Format

```json
{
  "<repo-name>": {
    "token": "github_pat_...",
    "owner": "<org-or-user>"
  }
}
```

| Field | Description |
|---|---|
| `<repo-name>` | Exact GitHub repo name — used as both the lookup key and in the dispatch URL |
| `token` | GitHub Personal Access Token (classic) with `repo` scope |
| `owner` | GitHub org name or username that owns the repo |

The `repo` field is **not** stored inside the entry. The key itself is the repo name and is
inserted directly into the endpoint URL:
`https://api.github.com/repos/{owner}/{key}/dispatches`

---

## Current value (single team)

```json
{
  "servicenow-issue-automation": {
    "token": "github_pat_11A4TTP6I0pVPVtRXHzPEg_...",
    "owner": "thev1ndu"
  }
}
```

---

## Multi-team example

If a second team uses a different repo on the same ServiceNow instance, add a second key:

```json
{
  "servicenow-issue-automation": {
    "token": "github_pat_...",
    "owner": "thev1ndu"
  },
  "platform-ops-issues": {
    "token": "github_pat_...",
    "owner": "another-org"
  }
}
```

Each team's flows use a different `REPO` constant — see [How flows use this](#how-flows-use-this).

---

## How flows use this

Every Script Step 2 in every flow has one constant at the top:

```javascript
var REPO = 'servicenow-issue-automation';
```

The rest of the lookup is identical across all flows:

```javascript
var configJson = gs.getProperty('github.dispatch.config');
var REPO     = 'servicenow-issue-automation';
var config   = JSON.parse(configJson)[REPO];
if (!config) {
  gs.error('No config entry for repo "' + REPO + '" in github.dispatch.config');
  outputs.http_status = 'config_missing';
  outputs.success     = 'false';
  return;
}
var endpoint = 'https://api.github.com/repos/' + config.owner + '/' + REPO + '/dispatches';
```

To point a flow at a different repo, change only the `REPO` constant on line 1.

---

## How to update the property in ServiceNow

1. Go to: `All > System Properties > System Properties`
2. Search for `github.dispatch.config`
3. Click the record
4. Edit the **Value** field — the full JSON object
5. Check **Ignore cache** so the updated value takes effect immediately
6. Click **Update**

> **Tip:** Use a JSON formatter before pasting to catch syntax errors.
> A malformed value will cause all flows to log `config_missing` and stop sending.

---

## How to add a new team / repo

1. Open `github.dispatch.config` as above
2. Add a new key to the JSON object:

```json
{
  "servicenow-issue-automation": { ... },
  "new-team-repo": {
    "token": "github_pat_...",
    "owner": "new-org"
  }
}
```

3. In the new team's Flow Designer flows, set `var REPO = 'new-team-repo';` in each Script Step 2
4. The same REST Message (`GitHub Integration`) is shared — no new REST Message needed

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `config_missing` in flow logs | Property does not exist or JSON is invalid | Verify the property exists and value is valid JSON |
| `No config entry for repo "..."` in flow logs | `REPO` constant does not match any key in the JSON | Check the key name matches exactly (case-sensitive) |
| `http_status: 401` | Token is expired or missing `repo` scope | Regenerate the PAT and update `token` in the property |
| `http_status: 404` | `owner` is wrong or repo does not exist | Check `owner` and the key name against the GitHub URL |
| Changes not taking effect | Property is cached | Enable **Ignore cache** on the property record |
