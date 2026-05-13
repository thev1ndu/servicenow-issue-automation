# GitHub ↔ ServiceNow Integration

A fully automated bridge between GitHub Issues and ServiceNow Customer Service Cases. Engineers raise Service Requests (SRs) directly in GitHub — validation, case creation, Change Request tracking, and comment synchronisation all happen automatically without anyone touching ServiceNow manually.

---

## Table of Contents

- [High-Level Idea](#high-level-idea)
- [System Architecture](#system-architecture)
- [How It Works — End-to-End](#how-it-works--end-to-end)
  - [Phase 1 — Raising a Service Request](#phase-1--raising-a-service-request)
  - [Phase 2 — Validation](#phase-2--validation)
  - [Phase 3 — Case Creation](#phase-3--case-creation)
  - [Phase 4 — Case Updates](#phase-4--case-updates)
  - [Phase 5 — Change Request Tracking](#phase-5--change-request-tracking)
  - [Phase 6 — Case Closure](#phase-6--case-closure)
  - [Phase 7 — Comment Synchronisation](#phase-7--comment-synchronisation)
- [Workflow Reference](#workflow-reference)
- [Issue Templates](#issue-templates)
- [Labels](#labels)
- [Configuration](#configuration)
- [GitHub Secrets](#github-secrets)
- [User Guide](#user-guide)
- [Bot Comments Reference](#bot-comments-reference)
- [Troubleshooting](#troubleshooting)

---

## High-Level Idea

Before this integration, engineers had to maintain two parallel records: a GitHub issue for team visibility and a ServiceNow case for ITSM tracking. The two systems drifted apart constantly — comments were lost, CR states were missed, and updates had to be duplicated by hand.

This integration eliminates that duplication. GitHub becomes the single interface for engineers. ServiceNow remains the authoritative ITSM record. Automation keeps both in sync.

```
Engineers work entirely in GitHub.
ServiceNow agents work entirely in ServiceNow.
Nothing falls through the gap.
```

**What the automation handles:**

| Trigger | Automated action |
|---------|-----------------|
| GitHub issue created from an SR template | Validates fields, creates a ServiceNow Case |
| Issue edited before any CR exists | PATCHes the ServiceNow Case with updated values |
| ServiceNow creates a Change Request | Posts a CR notification comment on the GitHub issue |
| CR changes state | Posts a state-change comment on the GitHub issue |
| Engineer comments on the GitHub issue | Copies the comment to the ServiceNow case |
| ServiceNow agent posts a case comment | Posts it as a comment on the GitHub issue |

---

## System Architecture

```mermaid
graph TB
    subgraph GitHub["GitHub"]
        GI[GitHub Issue]
        GA[GitHub Actions]
        GC[Issue Comments]
        GL[Labels]
    end

    subgraph Workflows["GitHub Actions Workflows"]
        W1[issue-servicenow.yml\nMain Orchestrator]
        W2[servicenow-create-case.yml\nCreate Case]
        W3[servicenow-update-case.yml\nUpdate Case]
        W4[sn-cr-notifier.yml\nCR Notifications]
        W5[github-comment-to-sn.yml\nComment Sync →]
        W6[sn-comment-to-github.yml\nComment Sync ←]
    end

    subgraph ServiceNow["ServiceNow"]
        SNC[Customer Service Case]
        SNCR[Change Request]
        SNBR[Business Rules]
        SNFD[Flow Designer]
        SNAPI[Scripted REST API]
    end

    GI -->|opened / edited / labeled| W1
    W1 -->|calls| W2
    W1 -->|calls| W3
    W2 -->|POST /case| SNAPI
    W3 -->|PATCH /case| SNAPI
    SNAPI --> SNC
    SNC --> SNCR
    SNBR -->|repository_dispatch| W4
    SNFD -->|repository_dispatch| W4
    SNFD -->|repository_dispatch| W6
    W4 -->|createComment| GC
    W6 -->|createComment| GC
    GC -->|issue_comment created| W5
    W5 -->|PATCH comments| SNC

    style GitHub fill:#f6f8fa,stroke:#d0d7de
    style Workflows fill:#fff8e1,stroke:#f0b429
    style ServiceNow fill:#e8f5e9,stroke:#43a047
```

---

## How It Works — End-to-End

### Phase 1 — Raising a Service Request

```mermaid
flowchart TD
    A([User opens GitHub Issues tab]) --> B[Clicks New Issue]
    B --> C{Pick a template}
    C -->|Complex production incident| D[SR Generic Requests\n14 required fields]
    C -->|Log retrieval| E[SR Request Logs\n7 fields · Critical priority only]
    C -->|Routine request| F[SR Standard Generic\n5 fields]
    C -->|Information only| G[SR Information\n3 fields]
    D & E & F & G --> H[Fill every required field\nNo blank _No response_ values]
    H --> I[Set title to\n'SR-Change: your summary']
    I --> J[Add one SRType label\nand one CatalogueItem label]
    J --> K([Automation starts automatically])

    style A fill:#e3f2fd
    style K fill:#e8f5e9
```

> **Why two labels?** `SRType/*` identifies the ITSM change type (Normal / Standard / Emergency). `CatalogueItem/*` tells the validator which field ruleset to apply. Both must be present before automation runs.

---

### Phase 2 — Validation

The main orchestrator [`issue-servicenow.yml`](.github/workflows/issue-servicenow.yml) runs on every `issues: [opened, edited, labeled]` event. A concurrency group (`sn-pipeline-<issue_number>`) prevents races when multiple events fire in quick succession.

```mermaid
flowchart TD
    T([Issue event fires]) --> G1{SR template marker\nin body?}
    G1 -->|No| EXIT1([Exit silently\nnot an SR issue])
    G1 -->|Yes| G2{SRType and\nCatalogueItem labels\nboth present?}
    G2 -->|No| EXIT2([Exit silently\nawaiting labels])
    G2 -->|Yes| V1{Title starts with\n'SR-Change:' ?}
    V1 -->|No| FAIL1[Post failure comment\n'Title must start with SR-Change:']
    FAIL1 --> STRIP[Remove SR labels\nand validation-passed]
    STRIP --> EXIT3([Exit])
    V1 -->|Yes| V2{All required fields\nfilled per ruleset?}
    V2 -->|No| FAIL2[Post failure comment\nlisting missing fields]
    FAIL2 --> STRIP
    V2 -->|Yes| V3{Extra rules\ne.g. Priority=Critical\nfor Request Logs?}
    V3 -->|Fail| FAIL3[Post failure comment\n'Priority must be Critical']
    FAIL3 --> STRIP
    V3 -->|Pass| PASS[Add validation-passed label\nPost success comment]
    PASS --> ROUTE{Case already exists?}
    ROUTE -->|No| CREATE([Call servicenow-create-case.yml])
    ROUTE -->|Yes and no CRs yet| UPDATE([Call servicenow-update-case.yml])
    ROUTE -->|Yes and CRs exist| LOCK([Skip — editing locked])

    style T fill:#e3f2fd
    style CREATE fill:#e8f5e9
    style UPDATE fill:#fff3e0
    style LOCK fill:#fce4ec
    style EXIT1 fill:#f5f5f5
    style EXIT2 fill:#f5f5f5
```

**Validation rules by catalogue item:**

| CatalogueItem | Required fields | Extra rule |
|---------------|----------------|------------|
| Generic Requests | Short Description, Description, Priority, Impact, Impact Description (Overall), Impact Description (Customer), Environment Details, Affected Component, Affected Services, Service Outage/Downtime, Is a maintenance window required, Implementation Plan, Test Plan, Monitoring Checks | — |
| Request Logs | Short Description, Description, Priority, Impact, Impact Description, Customer Project, Environment Details | Priority must be `Critical` |
| Standard Generic | Short Description, Description, Priority, Impact, Environment Details | — |
| Information | Request Description, Impact, Customer Project | — |

A field is blank if its value is `_No response_` (the GitHub template default) or empty.

---

### Phase 3 — Case Creation

[`servicenow-create-case.yml`](.github/workflows/servicenow-create-case.yml) is a reusable workflow called by the orchestrator after successful validation.

```mermaid
sequenceDiagram
    participant O as issue-servicenow.yml
    participant C as servicenow-create-case.yml
    participant GH as GitHub API
    participant SN as ServiceNow REST API

    O->>C: workflow_call (issue_number, catalog_item, sr_type)
    C->>C: Checkout repo
    C->>C: Load .github/servicenow-config.yml\n(account, project, product, wso2_product)
    C->>GH: GET issue body, title, labels
    GH-->>C: Issue data
    C->>C: Parse ### Section headers\nMap to SN field names
    C->>C: Build JSON payload\n(issue fields + config constants + metadata)
    C->>SN: POST /api/<scope>/github_case/v1/case\nBasic Auth
    SN-->>C: 201 Created\n{ case_number, sys_id }
    C->>GH: createComment — success\n✅ ServiceNow Case Created — [CS0001234](link)
    C->>GH: createComment — watching\n👀 Watching for Change Requests
    Note over C,GH: On failure: ❌ ServiceNow Case Creation Failed
```

**Payload built:** Every `### Section Name` in the issue body is extracted and mapped to its ServiceNow field name (e.g. `Short Description` → `u_short_description`). Config constants (`account`, `project`, `product`) are merged in. Issue metadata (number, URL, author, labels) is appended.

---

### Phase 4 — Case Updates

When an issue is edited **after** case creation **and before any CR has been created**, [`servicenow-update-case.yml`](.github/workflows/servicenow-update-case.yml) is called instead.

```mermaid
sequenceDiagram
    participant O as issue-servicenow.yml
    participant U as servicenow-update-case.yml
    participant GH as GitHub API
    participant SN as ServiceNow REST API

    O->>O: Detect: caseDone=true, isEdit=true, crCreated=false
    O->>U: workflow_call (issue_number)
    U->>GH: GET updated issue body
    GH-->>U: New field values
    U->>U: Re-extract all fields
    U->>U: Build PATCH payload
    U->>SN: PATCH /api/<scope>/github_case/v1/case
    SN-->>U: 200 OK\n{ changes: N, case_number, sys_id }
    U->>GH: createComment\n🔄 ServiceNow Case Updated · N field(s) updated
    Note over O: Once a CR exists, this path\nis permanently skipped
```

> **Edit lock:** Once a Change Request is detected in the issue comments, editing is permanently disabled for that issue. This prevents the GitHub issue from overwriting SN state after work has begun.

---

### Phase 5 — Change Request Tracking

ServiceNow Business Rules fire `repository_dispatch` events to GitHub. [`sn-cr-notifier.yml`](.github/workflows/sn-cr-notifier.yml) handles three action types.

```mermaid
sequenceDiagram
    participant SN as ServiceNow\nBusiness Rule
    participant GH_API as GitHub\nrepository_dispatch API
    participant W as sn-cr-notifier.yml
    participant SNTA as ServiceNow\nTable API
    participant GI as GitHub Issue

    SN->>GH_API: POST /repos/:owner/:repo/dispatches\nevent_type: servicenow-cr-update\nclient_payload: { action, cr_number, cr_state, ... }

    alt action = "created"
        GH_API->>W: Trigger workflow
        W->>SNTA: GET /api/now/table/task\n?parent=<case_sys_id>
        SNTA-->>W: All CRs for the case
        W->>GI: createComment\n💬 ServiceNow Change Request — Created #CHG...\nAll CRs listed
    else action = "state_changed"
        GH_API->>W: Trigger workflow
        W->>W: Resolve numeric state → display name\n(-5=New, -4=Assess, ... 4=Closed)
        W->>GI: createComment\n💬 ServiceNow Change Request — Updated\nPrevious → New state
    else action = "dates_updated"
        GH_API->>W: Trigger workflow
        W->>GI: createComment\n💬 Schedule Updated\nChange Window: start → end
    end
```

**CR State Map:**

```mermaid
stateDiagram-v2
    [*] --> New: CR created (-5)
    New --> Assess: -4
    Assess --> Authorize: -3
    Authorize --> CustomerApproval: -2
    CustomerApproval --> Scheduled: -1
    Scheduled --> Implement: 0
    Implement --> Review: 1
    Review --> CustomerReview: 2
    CustomerReview --> Closed: 4
    Implement --> Rollback: 3
    Rollback --> Closed: 4
    Scheduled --> Canceled: 5
    Authorize --> Canceled: 5
    Closed --> [*]
    Canceled --> [*]
```

---

### Phase 6 — Case Updates (Closed & Assigned)

A single Flow Designer flow handles both case closure and assignment. It sends a `servicenow-case-update` `repository_dispatch` event with an `action` field. [`sn-case-updates.yml`](.github/workflows/sn-case-updates.yml) branches on that action.

```mermaid
sequenceDiagram
    participant SN as ServiceNow\nFlow Designer
    participant GH_API as GitHub\nrepository_dispatch API
    participant W as sn-case-updates.yml
    participant GI as GitHub Issue

    Note over SN: Case state → Closed
    SN->>GH_API: event_type: servicenow-case-update\nclient_payload: { action: closed, github_issue_number,\ncase_number, case_sys_id, sn_user, resolution_notes }
    GH_API->>W: Trigger workflow
    W->>GI: createComment — ✅ ServiceNow Case Closed
    W->>GI: issues.update — state: closed, state_reason: completed

    Note over SN: Assigned to changes
    SN->>GH_API: event_type: servicenow-case-update\nclient_payload: { action: assigned, github_issue_number,\ncase_number, case_sys_id, assigned_to, sn_user }
    GH_API->>W: Trigger workflow
    W->>GI: createComment — 👤 ServiceNow Case Assigned
```

**Payload sent by ServiceNow:**

| Key | Required | Value |
|-----|----------|-------|
| `action` | Yes | `"closed"` or `"assigned"` |
| `github_issue_number` | Yes | GitHub issue number |
| `case_number` | Yes | ServiceNow case number e.g. `CS0001234` |
| `case_sys_id` | Yes | SN record sys_id (used to build the clickable link) |
| `sn_user` | No | Display name of the agent who made the change |
| `resolution_notes` | No | Closure summary (`closed` action only) |
| `assigned_to` | No | Display name of the person assigned (`assigned` action only) |

> **ServiceNow side:** See [GUIDE/ServiceNow/sn-case-updates.md](GUIDE/ServiceNow/sn-case-updates.md) for the single Flow Designer flow that handles both triggers.

---

### Phase 7 — Comment Synchronisation

Both directions of comment sync operate independently.

```mermaid
flowchart LR
    subgraph GH["GitHub"]
        GHU[Engineer posts comment\non GitHub issue]
        GHB[Bot comment posted\non GitHub issue]
    end

    subgraph WF["GitHub Actions"]
        W5[github-comment-to-sn.yml\nTriggered: issue_comment created]
        W6[sn-comment-to-github.yml\nTriggered: repository_dispatch\nservicenow-note]
    end

    subgraph SN["ServiceNow"]
        SNA[Case Additional Comments]
        SNFD[Flow Designer\n'SN Comment to GitHub' flow]
        SNAG[ServiceNow agent\nposts comment on case]
    end

    GHU -->|issue_comment: created\nnon-bot authors only| W5
    W5 -->|Parse sys_id from\n'Case Created' comment| W5
    W5 -->|PATCH sn_customerservice_case/<sys_id>\ncomments field| SNA

    SNAG --> SNFD
    SNFD -->|repository_dispatch\nservicenow-note| W6
    W6 -->|Echo loop guard:\nskip if note contains GitHub Comment| W6
    W6 -->|createComment\n💬 ServiceNow Comment| GHB

    style GH fill:#f6f8fa,stroke:#d0d7de
    style WF fill:#fff8e1,stroke:#f0b429
    style SN fill:#e8f5e9,stroke:#43a047
```

> **Echo loop prevention:** `sn-comment-to-github.yml` checks whether the incoming note contains `(GitHub Comment)` — the tag injected by `github-comment-to-sn.yml`. If it does, it skips posting, preventing an infinite reflection loop.

---

## Workflow Reference

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| [`issue-servicenow.yml`](.github/workflows/issue-servicenow.yml) | `issues: [opened, edited, labeled]` | Main orchestrator — validates and routes |
| [`servicenow-create-case.yml`](.github/workflows/servicenow-create-case.yml) | `workflow_call` | Builds payload and POSTs to ServiceNow |
| [`servicenow-update-case.yml`](.github/workflows/servicenow-update-case.yml) | `workflow_call` | PATCHes the case on issue edit |
| [`sn-cr-notifier.yml`](.github/workflows/sn-cr-notifier.yml) | `repository_dispatch: servicenow-cr-update` | Posts CR create / state-change / schedule comments |
| [`github-comment-to-sn.yml`](.github/workflows/github-comment-to-sn.yml) | `issue_comment: created` | Copies engineer comments to SN case |
| [`sn-comment-to-github.yml`](.github/workflows/sn-comment-to-github.yml) | `repository_dispatch: servicenow-note` | Posts SN agent comments on the issue |
| [`sn-case-updates.yml`](.github/workflows/sn-case-updates.yml) | `repository_dispatch: servicenow-case-update` | Posts assignment comments and closes the issue on case close |
| [`servicenow-inbound.yml`](.github/workflows/servicenow-inbound.yml) | `repository_dispatch: validation-passed` | Thin dispatcher for external/manual triggers |

### Concurrency

Both create and update workflows share the concurrency group `servicenow-issue-<issue_number>` with `cancel-in-progress: false`. This queues — never drops — concurrent runs for the same issue, preventing duplicate cases if a user applies both labels simultaneously.

---

## Issue Templates

Four templates are available under [`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/). Blank issue creation is disabled via [`config.yml`](.github/ISSUE_TEMPLATE/config.yml).

| Template | When to use | Required fields |
|----------|-------------|----------------|
| [SR Generic](.github/ISSUE_TEMPLATE/sr-generic.md) | Production incidents, complex changes with implementation and test plans | 14 fields |
| [SR Request Logs](.github/ISSUE_TEMPLATE/sr-request-logs.md) | Log extraction requests — Priority must be Critical | 7 fields |
| [SR Standard Generic](.github/ISSUE_TEMPLATE/sr-standard-generic.md) | Routine requests that don't need full plans | 5 fields |
| [SR Information](.github/ISSUE_TEMPLATE/sr-information.md) | Information-only requests; no change will be made | 3 fields |

Every template pre-fills the title as `[SR-Change]: ` and includes a hidden `### Template Marker` section that the orchestrator uses to detect SR issues.

---

## Labels

Labels are defined in [`.github/labels.yml`](.github/labels.yml) and must exist in the repository before any workflow runs.

```mermaid
graph LR
    subgraph SRType["SRType labels (blue)"]
        ST1[SRType/Normal Change]
        ST2[SRType/Standard Change]
        ST3[SRType/Emergency Change]
    end

    subgraph CatalogueItem["CatalogueItem labels (yellow)"]
        CI1[CatalogueItem/Generic Requests]
        CI2[CatalogueItem/Request Logs]
        CI3[CatalogueItem/Standard Generic]
        CI4[CatalogueItem/Information]
    end

    subgraph Auto["Automation labels"]
        VP[validation-passed\ngreen]
        VF[validation-failed\nred]
    end

    ST1 & ST2 & ST3 -->|One required| GATE[Orchestrator runs]
    CI1 & CI2 & CI3 & CI4 -->|One required| GATE
    GATE -->|Validation passes| VP
    GATE -->|Validation fails| VF
```

**Label application:**
- `SRType/*` — set by whoever processes the SR (identifies the ITSM change type)
- `CatalogueItem/*` — set by the requester to select the SR form type
- `validation-passed` / `validation-failed` — set automatically by workflows

---

## Configuration

[`.github/servicenow-config.yml`](.github/servicenow-config.yml) holds ServiceNow constants applied to every case. Edit once, commit — no secrets or repository variables needed.

```yaml
case:
  category: "Issue"                     # ServiceNow category display value
  account: "Your Account Name"          # Must exactly match the SN record display name
  announcement_type: "General"          # Exact dropdown value from SN

project:
  name: "Your Project Name"             # Project reference — exact display name
  product: "Your Product Name"          # Product reference — exact display name
  wso2_product: "Your WSO2 Product"     # WSO2 Product reference — exact display name
```

> **Name matching is exact.** For reference fields (`account`, `project`, `product`, `wso2_product`), a mismatch causes ServiceNow to silently leave the field blank — no error is returned. Copy the values character-for-character from the ServiceNow record.

---

## GitHub Secrets

Four secrets must be set in **Repository Settings → Secrets and variables → Actions** before any workflow can run.

| Secret | Value |
|--------|-------|
| `SERVICENOW_URL` | Full Scripted REST endpoint: `https://<instance>.service-now.com/api/<scope>/github_case/v1/case` |
| `SERVICENOW_UI_URL` | Base instance URL: `https://<instance>.service-now.com` — used to build clickable links |
| `SERVICENOW_USERNAME` | ServiceNow user with REST API access |
| `SERVICENOW_PASSWORD` | Password for the above user |

---

## User Guide

### Step 1 — Create a new issue

Go to the **Issues** tab and click **New issue**. Select the template that matches your request. Do not use blank issues (disabled).

### Step 2 — Fill in every required field

Replace every `_No response_` placeholder with real content. Leave no required field empty.

**Accepted priority values:** `Critical` · `High` · `Moderate` · `Low`  
**Accepted impact values:** `High` · `Medium` · `Low`

### Step 3 — Set the issue title

```
[SR-Change]: <short summary of your request>
```

Example: `[SR-Change]: Production API Gateway returning 502 errors`

The validation fails if the title does not start with `[SR-Change]:`.

### Step 4 — Add labels

Add exactly:
- One `SRType/*` label (`SRType/Normal Change`, `SRType/Standard Change`, or `SRType/Emergency Change`)
- One `CatalogueItem/*` label matching the template you used

Automation starts within seconds of both labels being present.

### Step 5 — Watch the comments

Every automated step posts a comment. No ServiceNow login needed to track progress.

```mermaid
sequenceDiagram
    actor U as You
    participant GI as GitHub Issue
    participant BOT as GitHub Actions Bot
    participant SN as ServiceNow Agent

    U->>GI: Create issue from template
    U->>GI: Fill fields, set title
    U->>GI: Add SRType + CatalogueItem labels
    BOT->>GI: ✅ Template Validation Passed
    BOT->>GI: ✅ ServiceNow Case Created — [CS0001234](link)
    BOT->>GI: 👀 Watching for Change Requests
    Note over SN: Agent reviews the case
    SN->>SN: Creates Change Request CHG0000001
    BOT->>GI: 💬 Change Request Created — CHG0000001 (New)
    Note over SN: Agent works the CR
    SN->>SN: Moves CR to Scheduled
    BOT->>GI: 💬 Change Request Updated — New → Scheduled
    SN->>SN: Implements, closes CR
    BOT->>GI: 💬 Change Request Updated — Implement → Closed
    U->>GI: Add a comment with questions
    BOT->>SN: Comment copied to SN case
    SN->>GI: 💬 ServiceNow Comment — Jane Smith replied
```

---

## Bot Comments Reference

| Comment | Workflow | When posted |
|---------|----------|------------|
| `✅ Template Validation Passed` | `issue-servicenow.yml` | All fields and labels valid |
| `❌ Template Validation Failed` | `issue-servicenow.yml` | Missing fields, bad title, or wrong priority |
| `✅ ServiceNow Case Created — [CSxxxxxxx]` | `servicenow-create-case.yml` | Case created in ServiceNow |
| `❌ ServiceNow Case Creation Failed` | `servicenow-create-case.yml` | POST to ServiceNow failed |
| `👀 Watching for Change Requests` | `issue-servicenow.yml` (cr-watch job) | Alongside case creation success |
| `🔄 ServiceNow Case Updated` | `servicenow-update-case.yml` | Issue edited, PATCH succeeded |
| `❌ ServiceNow Case Update Failed` | `servicenow-update-case.yml` | PATCH to ServiceNow failed |
| `💬 Change Request — Created #CHGxxxxxxx` | `sn-cr-notifier.yml` | SN Business Rule fires on CR insert |
| `💬 Change Request — Updated #CHGxxxxxxx` | `sn-cr-notifier.yml` | SN Business Rule fires on CR state change |
| `💬 Change Request — Schedule Updated` | `sn-cr-notifier.yml` | SN Business Rule fires on CR date change |
| `✅ ServiceNow Case Closed — [CSxxxxxxx]` | `sn-case-updates.yml` | SN case closed — GitHub issue is also closed |
| `👤 ServiceNow Case Assigned — [CSxxxxxxx]` | `sn-case-updates.yml` | SN case assigned to a team member |
| `💬 ServiceNow Comment — Case CSxxxxxxx` | `sn-comment-to-github.yml` | SN Flow Designer posts a case comment |
| `📋 ServiceNow Work Note` | `sn-comment-to-github.yml` | SN Flow Designer posts a work note |

---

## Troubleshooting

**Validation failed — what do I do?**

Read the failure comment. It names the specific field or rule that failed. Fix the issue body and re-add the `SRType/*` and `CatalogueItem/*` labels to re-trigger validation. The previous failure comment is automatically deleted when you retry.

**I edited the issue but no update comment appeared.**

Check whether a `💬 Change Request — Created` comment already exists on the issue. Once a CR exists, issue-edit updates are permanently disabled to prevent overwriting in-progress work. Post a comment instead — it will sync to the case.

**"ServiceNow Case Creation Failed" appeared.**

This is a configuration problem (wrong credentials, endpoint URL, or `servicenow-config.yml` values not matching SN display names). Check the workflow logs linked in the comment. Do not close and reopen the issue — fix the configuration and re-trigger by removing and re-adding the labels.

**The case was created but SN fields are blank.**

Reference fields (`account`, `project`, `product`, `wso2_product`) in `servicenow-config.yml` must exactly match ServiceNow display names. A mismatch causes silent blank — no error is returned by ServiceNow. Copy the values character-for-character from the SN record.

**CR state changes are not appearing on the issue.**

The ServiceNow Business Rules must be configured to send `repository_dispatch` events to GitHub. Verify the `github.dispatch.config` system property in ServiceNow contains the correct PAT, owner, and repo. See `GUIDE/ServiceNow/sn-cr-notifier.md` for setup instructions.

**Comments are not syncing to ServiceNow.**

`github-comment-to-sn.yml` only runs when a `✅ ServiceNow Case Created` comment exists on the issue (it reads the `sys_id` from the URL in that comment). If case creation failed, comment sync will not work.

**Do I need a ServiceNow account?**

No. All case and CR information is visible on the GitHub issue via bot comments. The comments include direct links to ServiceNow records if you need to view them in detail (ServiceNow credentials required to log in there).

---

## Repository File Map

```
.github/
├── workflows/
│   ├── issue-servicenow.yml          Main orchestrator
│   ├── servicenow-create-case.yml    Reusable: POST case to SN
│   ├── servicenow-update-case.yml    Reusable: PATCH case on issue edit
│   ├── sn-cr-notifier.yml            Inbound: CR create/state/schedule events
│   ├── github-comment-to-sn.yml      Outbound: engineer comments → SN
│   ├── sn-case-updates.yml           Inbound: SN case closed / assigned → GitHub
│   ├── sn-comment-to-github.yml      Inbound: SN agent comments → GitHub
│   └── servicenow-inbound.yml        Thin dispatcher for manual triggers
├── ISSUE_TEMPLATE/
│   ├── config.yml                    Disables blank issues
│   ├── sr-generic.md                 Full 14-field SR form
│   ├── sr-request-logs.md            Log request form (Critical only)
│   ├── sr-standard-generic.md        Simplified 5-field form
│   └── sr-information.md             Minimal 3-field information form
├── labels.yml                        Label definitions
└── servicenow-config.yml             ServiceNow constants (account, project, product)
```
