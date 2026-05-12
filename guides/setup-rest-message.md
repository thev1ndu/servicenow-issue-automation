# Guide: Create the GitHub Integration REST Message (Do this once)

This REST Message is shared by both flows:
- `SN Comment to GitHub` — sends comments/work notes to GitHub
- `SN CR Created → GitHub` and `SN CR State Changed → GitHub` — sends CR notifications

Both flows call this same REST Message with different HTTP Methods.

---

## Step 1 — Navigate to REST Messages

Go to: `All > System Web Services > Outbound > REST Messages`

Click **New** (top right of the list).

---

## Step 2 — Fill in the REST Message record

You will see a form with these fields:

| Field | What to enter |
|---|---|
| **Name** | `GitHub Integration` |
| **Application** | Leave as `Global` |
| **Accessible from** | Change to `All application scopes` so both flows can call it |
| **Description** | `Sends repository_dispatch events to GitHub Actions` |
| **Endpoint** | `https://api.github.com/repos/YOUR_ORG/YOUR_REPO/dispatches` |

Replace `YOUR_ORG` and `YOUR_REPO` with your actual GitHub organisation and repository name.
Example: `https://api.github.com/repos/wso2/support-automation/dispatches`

---

## Step 3 — Set Authentication

Click the **Authentication** tab (already visible on the form).

- **Authentication type**: Select `Basic`
- **Basic auth profile**: Click the search icon and select `Github`
  (This is the existing Basic Auth Configuration in your instance — visible in the lookup popup)

> The `Github` profile already stores the GitHub username and PAT.
> ServiceNow will automatically add the `Authorization: Basic ...` header when the request is sent.
> Do NOT add an Authorization header manually anywhere — the profile handles it.

Click **Submit** to save the REST Message.

---

## Step 4 — Add HTTP Method: `dispatch` (for comments)

After submitting, you are back on the saved REST Message record.

Scroll down to the **HTTP Methods** related list and click **New**.

### 4.1 Basic fields

| Field | Value |
|---|---|
| **Name** | `dispatch` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave blank — it inherits the URL from the parent REST Message |

Click **Save** (stay on the HTTP Method record).

### 4.2 Add HTTP Headers

Click the **HTTP Request** tab. You will see an **HTTP Headers** section with "Insert a new row...".

Click **Insert a new row...** and add these two headers one at a time:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

Click **Save** after adding each row, or click outside the row to confirm it.

### 4.3 Add the Request Body

Still on the **HTTP Request** tab, scroll down to the **Content** section.

In the **Request body** field, paste exactly:

```
{"event_type":"servicenow-note","client_payload":{"issue_number":"${issue_number}","note_text":"${note_text}","note_type":"${note_type}","case_number":"${case_number}","sn_user":"${sn_user}"}}
```

> The `${variable_name}` placeholders are filled in by the calling script before the request is sent.

### 4.4 Add Variable Substitutions

Scroll down to the **Variable Substitutions** related list. Click **New** for each row below:

| Name | Test value |
|---|---|
| `issue_number` | `42` |
| `note_text` | `Test comment from ServiceNow` |
| `note_type` | `additional_comments` |
| `case_number` | `CS0000001` |
| `sn_user` | `John Smith` |

After adding all 5, click **Update** on the HTTP Method record to save everything.

---

## Step 5 — Add HTTP Method: `dispatch_cr` (for Change Request events)

Go back to the `GitHub Integration` REST Message (use the breadcrumb at the top).

Scroll to **HTTP Methods** and click **New** again.

### 5.1 Basic fields

| Field | Value |
|---|---|
| **Name** | `dispatch_cr` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave blank |

Click **Save**.

### 5.2 Add HTTP Headers

Same as Step 4.2 — add both headers:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

### 5.3 Add the Request Body

In the **Request body** field paste:

```
{"event_type":"servicenow-cr-update","client_payload":{"github_issue_number":"${issue_number}","cr_number":"${cr_number}","cr_sys_id":"${cr_sys_id}","cr_state":"${cr_state}","previous_state":"${previous_state}","cr_environment":"${cr_environment}","case_sys_id":"${case_sys_id}","action":"${action}"}}
```

### 5.4 Add Variable Substitutions

| Name | Test value |
|---|---|
| `issue_number` | `42` |
| `cr_number` | `CHG0012345` |
| `cr_sys_id` | `abc123def456abc123def456abc12345` |
| `cr_state` | `Assess` |
| `previous_state` | *(leave blank)* |
| `cr_environment` | `Production` |
| `case_sys_id` | `xyz789xyz789xyz789xyz789xyz78901` |
| `action` | `created` |

Click **Update**.

---

## Step 6 — Test the connection

On the `dispatch` HTTP Method record, click the **Test** link or button (usually found near the top of the record or in the related links section).

ServiceNow will fire a test request using the test values you entered. Check:
- HTTP Status returned should be `204` (GitHub's success response for repository_dispatch)
- If you get `401` — the Github Basic Auth profile has the wrong PAT or username
- If you get `404` — the org/repo URL in the Endpoint is wrong
- If you get `422` — the `event_type` value is not registered in any GitHub Actions workflow on the default branch

---

## Summary — what you now have

| REST Message | HTTP Method | Used by |
|---|---|---|
| `GitHub Integration` | `dispatch` | SN Comment to GitHub flow |
| `GitHub Integration` | `dispatch_cr` | SN CR Created and State Changed flows |

Both methods use the `Github` Basic Auth profile automatically.
Authentication, endpoint, and variable substitutions are all configured.

You can now proceed to the flow guides:
- [sn-comment-to-github.md](sn-comment-to-github.md)
- [sn-cr-notifier.md](sn-cr-notifier.md)
