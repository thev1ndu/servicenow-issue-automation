# Implementation

This folder documents the GitHub ↔ ServiceNow integration that was built for this repository.

## Documents in this folder

| File | What it covers |
|------|---------------|
| [SERVICENOW.md](SERVICENOW.md) | Every component configured inside ServiceNow: custom columns, system property (`github.dispatch.config`), Scripted REST API, Script Include, REST Message, and Flow Designer flows |
| [GITHUB_ACTIONS.md](GITHUB_ACTIONS.md) | Every file introduced on the GitHub side: workflows, issue templates, labels, and static config |
| [USER_GUIDE.md](USER_GUIDE.md) | How the system works from a user's perspective — how to raise an SR, what happens automatically, and how to track progress |

---

## What was implemented

The integration has two directions:

**GitHub → ServiceNow (inbound)**
A GitHub issue created from an SR template is validated by GitHub Actions and, if it passes, a ServiceNow Case is automatically created via the Scripted REST API.

**ServiceNow → GitHub (outbound)**
When a ServiceNow agent creates a Change Request or changes its state, a Flow Designer flow calls GitHub's `repository_dispatch` API. The corresponding GitHub Actions workflow posts the update as a comment on the GitHub issue.

An optional third flow syncs ServiceNow case comments to the GitHub issue.

---

## ServiceNow — what was built and where to find the instructions

| Component | Instruction file |
|-----------|-----------------|
| Custom columns `u_github_issue_number` and `u_github_issue_url` on `sn_customerservice_case` | [GUIDE/step1.md](../GUIDE/step1.md) § 1.1 |
| System property `github.dispatch.config` (GitHub PAT, owner, repo) | [GUIDE/step1.md](../GUIDE/step1.md) § 1.2 |
| `.github/servicenow-config.yml` — ServiceNow constants (account, project, product) | [GUIDE/step1.md](../GUIDE/step1.md) § 1.3 |
| Scripted REST API `GitHub Case Integration` — POST `/case` (create) | [GUIDE/step2.md](../GUIDE/step2.md) § 2.3 |
| Scripted REST API `GitHub Case Integration` — PATCH `/case` (update on issue edit) | [GUIDE/step2.md](../GUIDE/step2.md) § 2.6 |
| Script Include `GitHubRepositoryDispatch` | [GUIDE/step3.md](../GUIDE/step3.md) § 3.1 |
| REST Message `GitHub Integration` with `dispatch` and `dispatch_cr` HTTP Methods | [GUIDE/ServiceNow/setup-rest-message.md](../GUIDE/ServiceNow/setup-rest-message.md) |
| Flow Designer — `SN CR Created → GitHub` (Flow A) | [GUIDE/ServiceNow/sn-cr-notifier.md](../GUIDE/ServiceNow/sn-cr-notifier.md) |
| Flow Designer — `SN CR State Changed → GitHub` (Flow B) | [GUIDE/ServiceNow/sn-cr-notifier.md](../GUIDE/ServiceNow/sn-cr-notifier.md) |
| Flow Designer — `SN Comment to GitHub` (optional) | [GUIDE/ServiceNow/sn-comment-to-github.md](../GUIDE/ServiceNow/sn-comment-to-github.md) |
| End-to-end testing and troubleshooting | [GUIDE/step4.md](../GUIDE/step4.md) |

---

## GitHub — what was built and where to find the instructions

All GitHub files live in this repository and are version-controlled. No external setup is required beyond adding four secrets to the repository.

| Component | Path |
|-----------|------|
| Main orchestrator workflow | [.github/workflows/issue-servicenow.yml](../.github/workflows/issue-servicenow.yml) |
| Reusable workflow — create case | [.github/workflows/servicenow-create-case.yml](../.github/workflows/servicenow-create-case.yml) |
| Reusable workflow — update case | [.github/workflows/servicenow-update-case.yml](../.github/workflows/servicenow-update-case.yml) |
| CR notification handler | [.github/workflows/sn-cr-notifier.yml](../.github/workflows/sn-cr-notifier.yml) |
| GitHub comment → SN case sync | [.github/workflows/github-comment-to-sn.yml](../.github/workflows/github-comment-to-sn.yml) |
| SN comment → GitHub issue sync | [.github/workflows/sn-comment-to-github.yml](../.github/workflows/sn-comment-to-github.yml) |
| Issue templates (4 SR forms) | [.github/ISSUE_TEMPLATE/](../.github/ISSUE_TEMPLATE/) |
| Label definitions | [.github/labels.yml](../.github/labels.yml) |
| ServiceNow constants config | [.github/servicenow-config.yml](../.github/servicenow-config.yml) |

**GitHub Secrets required:**

| Secret | Value |
|--------|-------|
| `SERVICENOW_URL` | `https://<instance>.service-now.com/api/<scope>/github_case/v1/case` |
| `SERVICENOW_UI_URL` | `https://<instance>.service-now.com` |
| `SERVICENOW_USERNAME` | ServiceNow user with REST API access |
| `SERVICENOW_PASSWORD` | Password for the above user |

---

## Implementation order

If setting this up from scratch, follow this order:

1. **[GUIDE/step1.md](../GUIDE/step1.md)** — Add custom columns, create `github.dispatch.config` system property, fill in `.github/servicenow-config.yml`
2. **[GUIDE/step2.md](../GUIDE/step2.md)** — Create the Scripted REST API with POST and PATCH resources
3. **[GUIDE/step3.md](../GUIDE/step3.md)** — Create the `GitHubRepositoryDispatch` Script Include
4. **[GUIDE/ServiceNow/setup-rest-message.md](../GUIDE/ServiceNow/setup-rest-message.md)** — Create the `GitHub Integration` REST Message
5. **[GUIDE/ServiceNow/sn-cr-notifier.md](../GUIDE/ServiceNow/sn-cr-notifier.md)** — Create Flow Designer flows for CR notifications
6. **[GUIDE/ServiceNow/sn-comment-to-github.md](../GUIDE/ServiceNow/sn-comment-to-github.md)** — Create the comment sync flow (optional)
7. Add the four GitHub secrets to the repository
8. **[GUIDE/step4.md](../GUIDE/step4.md)** — Run end-to-end tests
