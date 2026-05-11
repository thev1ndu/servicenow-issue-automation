## Step 2 — Flow Designer: dispatch when a Change Request is created

Goal: when ServiceNow creates a **Change Request** record, call GitHub `repository_dispatch` with:

- `event_type = "servicenow-cr-update"`
- `client_payload.action = "created"`

GitHub will then comment on the issue with “New Change Request Created”.

---

### 1) Decide where the GitHub issue number lives (required)

The Flow must know the matching issue number. Common options:

- Store it on the **case**: `sn_customerservice_case.u_github_issue_number`
- Copy it onto the **CR** when the CR is created: `change_request.u_github_issue_number`

You need one of these before CR creation events can reach the correct GitHub issue.

**Recommended (simplest):** add a field on the CR, e.g. `change_request.u_github_issue_number`, and ensure your CR creation process populates it.

---

### 2) Create the flow (click-by-click)

1. Open **Flow Designer** (search “Flow Designer” in the left nav filter).
2. Click **New**.
3. Select **Flow**.
4. Set:
   - **Name**: `GitHub - CR created (dispatch)`
   - **Run as**: *System User* (recommended for integrations)
5. Click **Submit** / **Save** (button text varies by UI).
6. On the flow canvas, click **Add Trigger**.
7. Choose trigger type **Record**.
8. Configure the trigger:
   - **Table**: `Change Request [change_request]`
   - **When**: `Created`
9. Click **Done** / **Save**.

---

### 3) Add the “Run Script” action (where it is)

On the flow canvas:

1. Click **Add Action** (the “+” under the trigger).
2. In the action search box, type **Run Script**.
3. Choose **Run Script** (it may appear under “Utilities”).
4. You will see a script editor with a function signature like `function execute(inputs, outputs) { ... }`.
5. Paste the script from the next section into that editor.

If your instance **does not show** “Run Script”:

- Install/enable Flow Designer plugins that include the Utilities spoke, **or**
- Use a **Script step** inside a **Custom Action**, **or**
- Use a **Subflow** with a script step (same code).

---

### 4) Configure action Inputs/Outputs (this is the important part)

When you open the **Run Script** action configuration, you will see **Inputs** and **Outputs** sections.

#### Inputs (required)

Create 1 input that passes the trigger record into the script:

- **Input name**: `current`
- **Type**: **Record** / **Reference** (wording depends on your instance)
- **Table**: `Change Request [change_request]`
- **Value** (Data pill): pick the trigger record pill:
  - Usually: `Trigger → Record`
  - Sometimes: `Trigger → Change Request` or `Trigger → Current`

After this mapping, the script can always read the record as:

- `inputs.current`

If you mistakenly map the **Case** record into the input (e.g. `sn_customerservice_case`), this CR-created flow won’t have CR fields like `number`, `state`, etc.

#### Outputs (recommended)

Create these outputs so you can see success/failure in Flow execution details:

- **Output name**: `ok` (Type: True/False)
- **Output name**: `status` (Type: Integer)
- **Output name**: `error` (Type: String) *(optional but helpful)*

---

### 5) Paste this script (CR created → dispatch)

**Before pasting:** update the 2 field names marked `CHANGE THIS`.

```javascript
(function execute(inputs, outputs) {
  // The trigger record is exposed as a “data pill” in most instances.
  // Depending on your ServiceNow version, it may be inputs.current or a named object.
  var cr = inputs.current || inputs.change_request || inputs.record || null;
  if (!cr) {
    gs.error('Flow: could not resolve Change Request record from inputs');
    outputs.ok = false;
    outputs.status = 0;
    outputs.error = 'No Change Request input record';
    return;
  }

  // CHANGE THIS: where you store the GitHub issue number on the CR
  var issueNum = (cr.u_github_issue_number || '') + '';
  if (!issueNum) {
    gs.info('Flow: CR has no GitHub issue number; skipping dispatch');
    outputs.ok = false;
    outputs.status = 0;
    outputs.error = 'CR has no GitHub issue number';
    return;
  }

  // CHANGE THIS (recommended): parent case sys_id so GitHub can list all CRs for the case.
  // If your CR points to the case using a different field, use that.
  var caseSysId = (cr.parent || '') + '';

  var payload = {
    github_issue_number: parseInt(issueNum, 10),
    action: 'created',
    cr_number: (cr.number || '') + '',
    cr_sys_id: (cr.sys_id || '') + '',
    cr_state: (cr.state || '') + '',
    cr_environment: (cr.u_environment || 'Development') + '',
    case_sys_id: caseSysId
  };

  var d = new GitHubRepositoryDispatch();
  var res = d.send('servicenow-cr-update', payload);

  outputs.ok = res.ok;
  outputs.status = res.status;
  outputs.error = res.ok ? '' : (res.body || '');
})(inputs, outputs);
```

---

### 6) Activate and test

1. Click **Activate** in Flow Designer.
2. Create a Change Request and make sure it has:
   - `u_github_issue_number` populated (or your chosen field)
   - `parent` populated with the case sys_id (or your chosen linkage field)
3. In GitHub → **Actions**, you should see a run of **ServiceNow inbound**.
4. In the GitHub issue, you should see a new comment:
   - “New Change Request Created”
   - CR link and state/environment

If there is no GitHub comment:

- Check the Flow execution details for the “Run Script” step logs.
- Verify `github.dispatch.config` token access (403 means token/SSO/permissions).
- Verify the payload includes:
  - `github_issue_number` (must be the real issue number)
  - `action: created`
  - `cr_number`

