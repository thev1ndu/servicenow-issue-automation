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

| Field | What to enter |
|---|---|
| **Name** | `GitHub Integration` |
| **Application** | Leave as `Global` |
| **Accessible from** | Change to `All application scopes` |
| **Description** | `Sends repository_dispatch events to GitHub Actions` |
| **Endpoint** | `https://api.github.com/repos/YOUR_ORG/YOUR_REPO/dispatches` |

Replace `YOUR_ORG` and `YOUR_REPO` with your actual values.
Example: `https://api.github.com/repos/wso2/support-automation/dispatches`

---

## Step 3 — Set Authentication on the REST Message

Click the **Authentication** tab on the REST Message form.

- **Authentication type**: `Basic`
- **Basic auth profile**: click the search icon → select `Github` from the list

> This is the existing Basic Auth Configuration already in your instance.
> ServiceNow automatically adds the Authorization header using this profile.
> Do NOT add an Authorization header manually anywhere.

Click **Submit** to save.

---

## Step 4 — Add HTTP Method: `dispatch` (used by the Comment flow)

After submitting you land back on the saved REST Message record.

Scroll down to the **HTTP Methods** related list and click **New**.

You will see the HTTP Method form — this is the same form shown in the screenshots above.

---

### 4.1 Fill in the top section of the HTTP Method form

The top part of the form has these fields (visible in screenshot 1):

| Field | What to enter |
|---|---|
| **REST Message** | Already filled — shows `GitHub Integration` (read only) |
| **Name** | `dispatch` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave **blank** — the method inherits the URL from the parent REST Message |
| **Content** | Paste the request body here — see section 4.3 below |

---

### 4.2 Authentication tab on the HTTP Method

Click the **Authentication** tab (visible in screenshot 1 at the bottom).

- **Authentication type**: Leave as `Inherit from parent`

> "Inherit from parent" means this HTTP Method will use the Basic auth (Github profile)
> that was set on the REST Message. You do not need to set anything here.

---

### 4.3 Paste the request body into the Content field

The **Content** field is on the main form, directly above the Authentication/HTTP Request tabs.
This is the JSON body that will be sent to GitHub.

Click inside the **Content** field and paste:

```
{"event_type":"servicenow-note","client_payload":{"issue_number":"${issue_number}","note_text":"${note_text}","note_type":"${note_type}","case_number":"${case_number}","sn_user":"${sn_user}"}}
```

> The `${variable_name}` placeholders will be filled in by the calling script before the request fires.

---

### 4.4 Add HTTP Headers

Click the **HTTP Request** tab (visible in screenshot 2).

You will see the **HTTP Headers** section with a `+` icon and "Insert a new row...".

Click the `+` or click directly on "Insert a new row..." and add these two headers:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

How to add each row:
1. Click "Insert a new row..."
2. A new editable row appears with a **Name** column and a **Value** column
3. Click in the Name cell, type the header name
4. Press Tab or click the Value cell, type the value
5. Click the green checkmark or click outside the row to save it
6. Repeat for the second header

> Leave **HTTP Query Parameters** empty — nothing goes there.
> The second **Content** field visible at the bottom of the HTTP Request tab is the same
> field as the one above the tabs. It should already show what you pasted in step 4.3.

---

### 4.5 Click Submit

Click the **Submit** button at the top right of the page.

You are now back on the REST Message record.

---

### 4.6 Add Variable Substitutions for `dispatch`

The variable substitutions tell ServiceNow which `${placeholder}` names exist in the body
so it knows to replace them at runtime.

On the saved HTTP Method record (`dispatch`), scroll down to the **Variable Substitutions** related list.

Click **New** for each row:

| Name | Test value (used when you click Test) |
|---|---|
| `issue_number` | `42` |
| `note_text` | `Test comment from ServiceNow` |
| `note_type` | `additional_comments` |
| `case_number` | `CS0000001` |
| `sn_user` | `John Smith` |

For each:
1. Click **New**
2. The **Name** field — type the variable name exactly as listed (no `${}` brackets)
3. The **Test value** field — type the example value
4. Click **Submit**

After adding all 5, you should see them listed in the Variable Substitutions related list.

---

## Step 5 — Add HTTP Method: `dispatch_cr` (used by the CR flows)

Go back to the `GitHub Integration` REST Message using the breadcrumb at the top of the page.

Scroll to **HTTP Methods** and click **New** again.

---

### 5.1 Fill in the top section

| Field | What to enter |
|---|---|
| **Name** | `dispatch_cr` |
| **HTTP method** | `POST` |
| **Endpoint** | Leave **blank** |
| **Content** | Paste the body below |

Paste into the **Content** field:

```
{"event_type":"servicenow-cr-update","client_payload":{"github_issue_number":"${issue_number}","cr_number":"${cr_number}","cr_sys_id":"${cr_sys_id}","cr_state":"${cr_state}","previous_state":"${previous_state}","cr_environment":"${cr_environment}","case_sys_id":"${case_sys_id}","action":"${action}"}}
```

---

### 5.2 Authentication tab

Leave **Authentication type** as `Inherit from parent` — same as the `dispatch` method.

---

### 5.3 HTTP Request tab — Add headers

Click the **HTTP Request** tab. Add the same two headers:

| Name | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Accept` | `application/vnd.github+json` |

Click **Submit**.

---

### 5.4 Add Variable Substitutions for `dispatch_cr`

On the saved `dispatch_cr` HTTP Method record, scroll to **Variable Substitutions** and click **New** for each:

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

Click **Submit** after each one.

---

## Step 6 — Test the connection

On the `dispatch` HTTP Method record, scroll to the bottom of the page.

Under **Related Links** you will see:
- Preview Script Usage
- Set HTTP Log level
- **Test**

Click **Test**.

ServiceNow sends a real HTTP request to GitHub using the test values you entered in the Variable Substitutions.

A response panel appears showing:
- **HTTP status**: should be `204` — this is GitHub's success response for `repository_dispatch`
- **Response body**: will be empty on success (204 has no body)

If you get a different status:

| Status | Cause | Fix |
|---|---|---|
| `401` | Wrong PAT or username in the Github Basic Auth profile | Update the profile record |
| `404` | Wrong org or repo name in the Endpoint URL | Fix the URL on the REST Message record |
| `422` | The `event_type` (`servicenow-note`) is not registered in any workflow on the default branch | Push the workflow file to the default branch |

---

## Summary

| REST Message | HTTP Method | Content body event_type | Used by |
|---|---|---|---|
| `GitHub Integration` | `dispatch` | `servicenow-note` | SN Comment to GitHub flow |
| `GitHub Integration` | `dispatch_cr` | `servicenow-cr-update` | SN CR Created and State Changed flows |

Authentication is handled automatically by the `Github` Basic Auth profile on the parent REST Message.
Both HTTP Methods inherit it — no auth config needed on the methods themselves.

You can now proceed to:
- [sn-comment-to-github.md](sn-comment-to-github.md)
- [sn-cr-notifier.md](sn-cr-notifier.md)
