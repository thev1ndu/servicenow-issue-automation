# System Overview Diagram

## What This System Does

This automation bridges **GitHub** (where engineers work) and **ServiceNow** (where the support team works). Instead of someone manually copying information between the two systems, everything syncs automatically — creating cases, updating statuses, syncing comments, and even closing tickets — without anyone having to touch both platforms.

---

## High-Level Flow

```mermaid
flowchart TD
    classDef github fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef sn fill:#fee2e2,stroke:#dc2626,color:#7f1d1d
    classDef auto fill:#dcfce7,stroke:#16a34a,color:#14532d
    classDef decision fill:#fef9c3,stroke:#ca8a04,color:#713f12
    classDef endpoint fill:#f3e8ff,stroke:#9333ea,color:#581c87

    Start(["👩‍💻 Engineer raises an issue in GitHub\n(Bug · Service Request · Planned Change)"]):::endpoint

    Template["📝 Fills in a structured template\nwith all the details needed"]:::github

    Validate{"🤖 Automation checks:\nAre all required fields filled?"}:::decision

    Fail["⚠️ Bot comments on the issue\nlisting exactly what's missing"]:::auto

    Created["📋 ServiceNow case created automatically\nNo manual entry needed"]:::auto

    Agent["🎫 Support agent picks up\nthe case in ServiceNow"]:::sn

    Comments["💬 Comments & notes sync both ways\nGitHub ↔ ServiceNow in real time\nEcho-loop protection prevents duplicates"]:::auto

    CR["📝 Agent creates a Change Request\nfor planned or complex work\n(with schedule & impact info)"]:::sn

    CRNotify["🔔 GitHub issue updated automatically\nwith CR number, status & planned schedule"]:::auto

    Resolve["✅ Agent resolves the case\nin ServiceNow"]:::sn

    Done(["🔒 GitHub issue closes automatically\nNo manual action needed"]):::endpoint

    Start --> Template
    Template --> Validate
    Validate -- "Fields missing" --> Fail
    Fail -- "Engineer updates & saves issue" --> Validate
    Validate -- "All fields present" --> Created
    Created --> Agent
    Agent <--> Comments
    Comments <--> Start
    Agent --> CR
    CR --> CRNotify
    CRNotify --> Start
    Agent --> Resolve
    Resolve --> Done
```

---

## Key Points to Know

| What happens | How it works |
|---|---|
| **3 issue types supported** | Bug/Incident, Service Request, Planned Change — each with its own required fields |
| **Validation gate** | If anything is missing, the bot tells you exactly what to fix before a case is created |
| **Automatic case creation** | Once valid, a ServiceNow case appears in the queue with all details pre-filled |
| **Bidirectional comments** | Anything you write in GitHub appears in ServiceNow, and vice versa |
| **Change Request tracking** | When a Change Request is raised in ServiceNow, GitHub is notified with the CR number and schedule |
| **Auto-close** | When the support agent closes the ServiceNow case, the GitHub issue closes itself |

---

## The Two Worlds

```
┌────────────────────────────────┐      ┌──────────────────────────────────┐
│         GITHUB SIDE            │      │        SERVICENOW SIDE           │
│                                │      │                                  │
│  Engineers raise & track       │◄────►│  Support agents triage & resolve │
│  issues using templates        │      │  cases in their normal workflow  │
│                                │      │                                  │
│  • Bug Reports                 │      │  • Case queue                    │
│  • Service Requests            │      │  • Change Requests               │
│  • Planned Changes             │      │  • Work notes & comments         │
│                                │      │  • Case closure                  │
└────────────────────────────────┘      └──────────────────────────────────┘
                         ▲                       ▲
                         └──────────┬────────────┘
                                    │
                         ⚙️ GitHub Actions Automation
                         (runs silently in the background)
```
