# Guide: ServiceNow Case Tags → GitHub Labels (Business Rule)

When an agent adds or removes `Change-Tracking/*` tags on a ServiceNow Case, this Business Rule
fires and syncs those tags to the linked GitHub issue as labels.

Supported tags (must exist on the SN instance):
- `Change-Tracking/Application`
- `Change-Tracking/Maintenance`
- `Change-Tracking/Infrastructure`

The BR only manages `Change-Tracking/*` labels on GitHub — all other labels on the issue are left untouched.

---

## How it works

1. Agent saves a Case after adding or removing a `Change-Tracking/*` tag
2. The After Business Rule fires
3. Script reads all `Change-Tracking/*` tags currently on the case from `sys_label_entry`
4. Calls GitHub to GET the current issue labels
5. Strips all existing `Change-Tracking/*` labels, keeps all others
6. Calls GitHub to PUT the merged label list back

The GET + PUT pattern ensures no other labels (e.g. `Type/Incident`, `validation-passed`) are disturbed.

---

## Before you start

- The GitHub labels must exist before they can be applied to issues.
  Add them via the `sync-labels` workflow or confirm `.github/labels.yml` includes:
  - `Change-Tracking/Application`
  - `Change-Tracking/Maintenance`
  - `Change-Tracking/Infrastructure`
- `github.dispatch.config` must have an entry for the case's Account — see [github-dispatch-config.md](github-dispatch-config.md)
- The case must have a value in `u_github_issue_number`
- The `Change-Tracking/*` tags must exist on the SN instance under **Tags** (`sys_label`)

---

## Create the Business Rule

Go to: `All > System Definition > Business Rules`

Click **New**.

---

### Basic fields

| Field | Value |
|---|---|
| **Name** | `SN Case Tags → GitHub Labels` |
| **Table** | `Customer Service Case [sn_customerservice_case]` |
| **Active** | ✅ checked |
| **Advanced** | ✅ checked |
| **When** | `after` |
| **Update** | ✅ checked |
| **Insert** | leave unchecked |

---

### Condition

In the **Condition** field (the single-line expression box near the top):

```javascript
current.sys_tags.changes() && current.u_github_issue_number != ''
```

This fires only when the tags field actually changed AND the case has a GitHub issue linked.

---

### Script

Paste into the **Script** area:

```javascript
(function executeRule(current, previous /*null when async*/) {

  var issueNumber = current.getValue('u_github_issue_number');
  if (!issueNumber) return;

  var accountName = current.account.getDisplayValue();
  if (!accountName) {
    gs.warn('SN Tags → GitHub: no account on case ' + current.number);
    return;
  }

  var configJson = gs.getProperty('github.dispatch.config');
  if (!configJson) {
    gs.error('SN Tags → GitHub: github.dispatch.config missing');
    return;
  }

  var config = JSON.parse(configJson)[accountName];
  if (!config) {
    gs.error('SN Tags → GitHub: no config entry for account "' + accountName + '"');
    return;
  }

  var baseUrl = 'https://api.github.com/repos/' + config.owner + '/' + config.repo;
  var authHeader = 'token ' + config.token;

  // Read all Change-Tracking/* tags currently saved on this case
  var tagGr = new GlideRecord('sys_label_entry');
  tagGr.addQuery('table', 'sn_customerservice_case');
  tagGr.addQuery('id', current.sys_id);
  tagGr.query();

  var changeTrackingLabels = [];
  while (tagGr.next()) {
    var labelName = tagGr.label.getDisplayValue();
    if (labelName.indexOf('Change-Tracking/') === 0) {
      changeTrackingLabels.push(labelName);
    }
  }

  // GET current GitHub issue labels
  var getRm = new sn_ws.RESTMessageV2();
  getRm.setHttpMethod('GET');
  getRm.setEndpoint(baseUrl + '/issues/' + issueNumber + '/labels');
  getRm.setRequestHeader('Authorization', authHeader);
  getRm.setRequestHeader('Accept', 'application/vnd.github+json');

  var getResp = getRm.execute();
  var keepLabels = [];

  if (getResp.getStatusCode() == 200) {
    var existing = JSON.parse(getResp.getBody());
    for (var i = 0; i < existing.length; i++) {
      // Keep every label that is NOT a Change-Tracking/* label
      if (existing[i].name.indexOf('Change-Tracking/') !== 0) {
        keepLabels.push(existing[i].name);
      }
    }
  } else {
    gs.warn('SN Tags → GitHub: GET labels returned HTTP ' + getResp.getStatusCode());
    return;
  }

  // Merge: kept labels + current Change-Tracking tags from SN
  var newLabels = keepLabels.concat(changeTrackingLabels);

  // PUT merged label list to GitHub
  var putRm = new sn_ws.RESTMessageV2();
  putRm.setHttpMethod('PUT');
  putRm.setEndpoint(baseUrl + '/issues/' + issueNumber + '/labels');
  putRm.setRequestHeader('Authorization', authHeader);
  putRm.setRequestHeader('Accept', 'application/vnd.github+json');
  putRm.setRequestHeader('Content-Type', 'application/json');
  putRm.setRequestBody(JSON.stringify({ labels: newLabels }));

  var putResp = putRm.execute();
  var httpStatus = putResp.getStatusCode();

  if (httpStatus !== 200) {
    gs.warn('SN Tags → GitHub: PUT labels failed. HTTP ' + httpStatus + ' Body: ' + putResp.getBody());
  }

})(current, previous);
```

Click **Submit**.

---

## Ensure the Change-Tracking tags exist in ServiceNow

Go to: `All > Tags`

Check that the following tags exist (create them if not):

| Tag name |
|---|
| `Change-Tracking/Application` |
| `Change-Tracking/Maintenance` |
| `Change-Tracking/Infrastructure` |

To create one: click **New**, enter the name, set **Visibility** to `Everyone`, click **Submit**.

---

## Testing

1. Open a Customer Service Case that has a value in `u_github_issue_number`
2. Scroll to the **Tags** field and add `Change-Tracking/Application`
3. Click **Save**
4. Open the linked GitHub issue — the label `Change-Tracking/Application` should appear within a few seconds
5. Go back to the case, remove `Change-Tracking/Application`, add `Change-Tracking/Infrastructure`, save
6. GitHub issue should now show `Change-Tracking/Infrastructure` and not `Change-Tracking/Application`
7. Verify that any other labels already on the issue (e.g. `Type/Incident`) were not removed

---

## Checking execution logs

Go to: `All > System Log > Business Rules`

Filter by **Name** = `SN Case Tags → GitHub Labels`. The log shows `gs.warn` / `gs.error` output from the script.

Alternatively: `All > System Log > All` → filter by message containing `SN Tags → GitHub`.

---

## Troubleshooting

| Problem | What to check |
|---|---|
| BR does not fire | Confirm **When** is `after`, **Update** is checked, and the Condition is saved correctly |
| `no config entry for account` | The case's Account display name doesn't match any key in `github.dispatch.config` |
| GET returns `401` | Token in `github.dispatch.config` is expired or missing `repo` scope |
| GET returns `404` | `owner` or `repo` in the config is wrong, or the issue number doesn't exist |
| PUT returns `422` | One of the label names doesn't exist on the GitHub repo — run the `sync-labels` workflow first |
| Tags not read correctly | Confirm `sys_label_entry` is populated — query it in Script Background: `var gr = new GlideRecord('sys_label_entry'); gr.addQuery('id', '<case_sys_id>'); gr.query(); while(gr.next()) { gs.info(gr.label.getDisplayValue()); }` |
| Other labels removed from issue | Check the GET response — if it returned non-200, `keepLabels` is empty. Investigate the GET failure first |
