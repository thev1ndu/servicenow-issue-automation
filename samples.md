## Samples: test GitHub Actions with payloads (no ServiceNow required)

This repo has two kinds of triggers:

1. **Issue label events** (GitHub native): `issue-servicenow.yml` runs when labels are added.
2. **ServiceNow → GitHub events** (webhook-style): `servicenow-inbound.yml` runs on `repository_dispatch`.

This document focuses on **testing `repository_dispatch`** using sample payloads, without doing anything in ServiceNow.

---

## Prerequisites

- A GitHub token that can call `repository_dispatch` for this repo:
  - Fine‑grained PAT: Repository access to this repo + **Contents: Read and write**
  - Classic PAT (private repo): `repo` scope
- Repo values:
  - `OWNER`: GitHub org/user
  - `REPO`: repository name
- A GitHub issue number you can post comments to:
  - `ISSUE_NUMBER`

Export these locally:

```bash
export OWNER="thev1ndu"
export REPO="servicenow-issue-automation"
export GH_TOKEN="YOUR_GITHUB_TOKEN"
export ISSUE_NUMBER="1"
```

---

## 1) Test: CR created → GitHub issue comment

This triggers `servicenow-inbound.yml` with `event_type=servicenow-cr-update` and `action=created`.

### Command

```bash
curl -i -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer ${GH_TOKEN}" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "https://api.github.com/repos/${OWNER}/${REPO}/dispatches" \
  -d @- <<JSON
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": ${ISSUE_NUMBER},
    "action": "created",
    "cr_number": "CHG0000001",
    "cr_sys_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "cr_state": "New",
    "cr_environment": "Development"
  }
}
JSON
```

### Expected result

- HTTP response: **204 No Content**
- GitHub Actions: a run of **ServiceNow inbound**
- GitHub issue: a new comment:
  - “New Change Request Created”
  - CR number + state + environment

**Note:** This sample intentionally omits `case_sys_id` so the workflow will not try to query ServiceNow for “all CRs”.

---

## 2) Test: CR state changed → GitHub issue comment

### Command

```bash
curl -i -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer ${GH_TOKEN}" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "https://api.github.com/repos/${OWNER}/${REPO}/dispatches" \
  -d @- <<JSON
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": ${ISSUE_NUMBER},
    "action": "state_changed",
    "cr_number": "CHG0000001",
    "cr_sys_id": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
    "previous_state": "New",
    "cr_state": "Assess"
  }
}
JSON
```

### Expected result

- GitHub issue gets a new comment:
  - “Change Request State Updated”
  - “State Change: New → Assess”

---

## 3) Test: CR created with optional counters

If your ServiceNow payload includes counters like `is_first_cr` / `total_crs`, the workflow will show them.

```bash
curl -i -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer ${GH_TOKEN}" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "https://api.github.com/repos/${OWNER}/${REPO}/dispatches" \
  -d @- <<JSON
{
  "event_type": "servicenow-cr-update",
  "client_payload": {
    "github_issue_number": ${ISSUE_NUMBER},
    "action": "created",
    "cr_number": "CHG0000002",
    "cr_sys_id": "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
    "cr_state": "New",
    "cr_environment": "Development",
    "is_first_cr": false,
    "total_crs": 2
  }
}
JSON
```

---

## 4) Troubleshooting `repository_dispatch`

### If `curl` returns 403

- Token can’t access the repo (wrong permissions, not authorized for org SSO, wrong repo selected in fine‑grained token).
- Fix:
  - Fine‑grained PAT: **Contents = Read and write** + repo selected
  - Org SSO: authorize the token for the org

### If `curl` returns 204 but you see no comment

- Confirm the workflow exists: `.github/workflows/servicenow-inbound.yml`
- Confirm required fields are present in `client_payload`:
  - `github_issue_number`, `action`, `cr_number`
- Confirm `github_issue_number` points to a real issue in the same repo.

---

## Optional: test the label-driven pipeline (no ServiceNow call)

The full pipeline creates a ServiceNow case, so it will fail without valid ServiceNow secrets.
However, you can still test that the **label trigger fires**:

1. Create an issue from `.github/ISSUE_TEMPLATE/sr-generic.md`
2. Add labels `SRType/...` and `CatalogueItem/...`
3. Confirm `Issue → ServiceNow` workflow starts (it may later fail on ServiceNow call if secrets are not set)

