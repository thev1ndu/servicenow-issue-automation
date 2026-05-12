# ServiceNow Setup Guide

Complete click-by-click instructions for configuring the ServiceNow side of the GitHub ↔ ServiceNow integration.

## What you are setting up

| Direction | What | How |
|-----------|------|-----|
| GitHub → ServiceNow | Create a Case from a GitHub issue | Scripted REST API (inbound POST /case) |
| ServiceNow → GitHub | Notify GitHub when a CR is created or changes state | Script Include + Business Rules |

## Steps

| File | What it covers |
|------|---------------|
| [step1.md](step1.md) | Prerequisites: custom columns, system property, `.github/servicenow-config.yml` |
| [step2.md](step2.md) | Scripted REST API — inbound Case creation endpoint (POST + PATCH) |
| [step3.md](step3.md) | Script Include + Business Rules — outbound CR notifications |
| [step4.md](step4.md) | End-to-end testing and troubleshooting |

Start with **step1.md** and work through them in order.

## Quick-start checklist

- [ ] Add `u_github_issue_number` and `u_github_issue_url` columns to `sn_customerservice_case` (step 1.1)
- [ ] Create `github.dispatch.config` system property with GitHub PAT (step 1.2)
- [ ] Fill in `.github/servicenow-config.yml` with your SN account, project, and product names (step 1.3)
- [ ] Create Scripted REST API `GitHub Case Integration` with POST `/case` resource (step 2.3)
- [ ] Add PATCH `/case` resource to the same API (step 2.6)
- [ ] Create Script Include `GitHubRepositoryDispatch` (step 3.1)
- [ ] Create Business Rule `Notify GitHub on CR created` (step 3.2)
- [ ] Create Business Rule `Notify GitHub on CR state change` (step 3.3)
- [ ] Set GitHub secrets: `SERVICENOW_URL`, `SERVICENOW_USERNAME`, `SERVICENOW_PASSWORD`, `SERVICENOW_UI_URL`
- [ ] Run end-to-end tests (step 4)

## Full reference

See [ServiceNow.md](../ServiceNow.md) at the repo root for all scripts and field definitions in one place.
