# Guide: Create the GitHub Integration REST Message (Do this once)

This REST Message is a lightweight container used by both flows.
The GitHub token, owner, and repo are **not** stored here — they are read at runtime
from the ServiceNow System Property `github.dispatch.config`.

Both flows call this same REST Message with different HTTP Methods.

---

## How the system property is used

The property `github.dispatch.config` holds a JSON object keyed by repo name:

```json
{
  "servicenow-issue-automation": {
    "token": "github_pat_...",
    "owner": "your-org"
  }
}
```

The Script step in each flow declares a `REPO` constant matching the key, looks up the entry, and:
- Builds the endpoint URL: `https://api.github.com/repos/{owner}/{REPO}/dispatches`
- Sets the `Authorization: token {token}` header

The repo name is the key — it is not stored inside the entry but used directly when building the endpoint URL.

Because of this, the REST Message itself needs **no auth** and **no variable substitutions**.
The script handles all of that at runtime.

See [github-dispatch-config.md](github-dispatch-config.md) for the full format and multi-repo setup.

---

## Step 1 — Navigate to REST Messages

Go to: `All > System Web Services > Outbound > REST Messages`

Click **New**.

---

## Step 2 — Fill in the REST Message record

| Field | What to enter |
|---|---|
| **Name** | `GitHub Integration` |
| **Application** | `Global` |
| **Accessible from** | `All application scopes` |
| **Description** | `Sends repository_dispatch events to GitHub Actions` |
| **Endpoint** | `https://api.github.com` |

> The endpoint here is just a placeholder. The script overrides it with the full URL
> (including owner/repo from the system property) before each request is sent.

---

## Step 3 — Authentication tab on the REST Message

Click the **Authentication** tab.

- **Authentication type**: Leave as `No authentication`

> Auth is handled by the script using the token from `github.dispatch.config`.
> Do not set a Basic Auth profile here.

Click **Submit**.

---

## Step 4 — Add HTTP Method: `dispatch` (used by the Comment flow)

On the saved REST Message record, scroll to **HTTP Methods** and click **New**.

### 4.1 Fill in the top section

| Field | What to enter |
|---|---|
| **Name** | `dispatch` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave **blank** |
| **Content** | Leave **blank** — the script sets the body directly |

### 4.2 Authentication tab on the HTTP Method

- **Authentication type**: `Inherit from parent`

> The parent has no auth — that is correct. The script adds the Authorization header at runtime.

### 4.3 HTTP Request tab — Add headers

Click the **HTTP Request** tab.

Under **HTTP Headers**, click "Insert a new row..." and add:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

> Do NOT add an Authorization header here.
> The script adds it from the system property token.

Leave **HTTP Query Parameters** empty.

Leave the **Content** field at the bottom of this tab blank.

Click **Submit**.

> No Variable Substitutions needed — the script builds and sets the body directly.

---

## Step 5 — Add HTTP Method: `dispatch_cr` (used by the CR flows)

Go back to the `GitHub Integration` REST Message. Scroll to **HTTP Methods** → **New**.

### 5.1 Fill in the top section

| Field | What to enter |
|---|---|
| **Name** | `dispatch_cr` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave **blank** |
| **Content** | Leave **blank** |

### 5.2 Authentication tab

- **Authentication type**: `Inherit from parent`

### 5.3 HTTP Request tab — Add headers

Same two headers as `dispatch`:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

Click **Submit**.

---

## Step 6 — Verify the system property exists

Go to: `All > System Properties > System Properties`

Search for `github.dispatch.config`.

The property value should be valid JSON keyed by repo name:

```json
{
  "servicenow-issue-automation": {
    "token": "github_pat_...",
    "owner": "thev1ndu"
  }
}
```

If the property does not exist:
1. Click **New**
2. **Name**: `github.dispatch.config`
3. **Type**: `String`
4. **Value**: paste the JSON above with your actual values
5. Click **Submit**

---

## Step 7 — Test using Script Background

Before activating any flow, test that the system property read and the REST call work correctly.

Go to: `All > System Definition > Scripts - Background`

Paste and run:

```javascript
var configJson = gs.getProperty('github.dispatch.config');
var REPO   = 'servicenow-issue-automation';
var config = JSON.parse(configJson)[REPO];

gs.info('Owner: ' + config.owner);
gs.info('Repo:  ' + REPO);
gs.info('Token starts with: ' + config.token.substring(0, 10));

var rm = new sn_ws.RESTMessageV2('GitHub Integration', 'dispatch');
rm.setEndpoint('https://api.github.com/repos/' + config.owner + '/' + REPO + '/dispatches');
rm.setRequestHeader('Authorization', 'token ' + config.token);
rm.setRequestBody(JSON.stringify({
  event_type: 'servicenow-note',
  client_payload: {
    issue_number: '1',
    note_text: 'Test from ServiceNow Script Background',
    note_type: 'additional_comments',
    case_number: 'CS0000001',
    sn_user: gs.getUserDisplayName()
  }
}));

var response = rm.execute();
gs.info('HTTP Status: ' + response.getStatusCode());
gs.info('Response: ' + response.getBody());
```

Check the output at the bottom of the page:
- `HTTP Status: 204` = success
- `401` = token in the system property is wrong or expired
- `404` = owner or repo in the system property is wrong
- `422` = the `sn-comment-to-github.yml` workflow is not on the default branch

---

## Summary

| REST Message | HTTP Method | Auth | Body |
|---|---|---|---|
| `GitHub Integration` | `dispatch` | Set by script at runtime | Set by script at runtime |
| `GitHub Integration` | `dispatch_cr` | Set by script at runtime | Set by script at runtime |

Everything dynamic (token, owner, repo, request body) comes from `github.dispatch.config`
and is applied by the calling script — not by the REST Message configuration.

You can now proceed to:
- [sn-comment-to-github.md](sn-comment-to-github.md)
- [sn-cr-notifier.md](sn-cr-notifier.md)
