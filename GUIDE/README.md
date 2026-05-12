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
| [step1.md](step1.md) | Prerequisites: custom columns, system property |
| [step2.md](step2.md) | Scripted REST API — inbound Case creation endpoint |
| [step3.md](step3.md) | Script Include + Business Rules — outbound CR notifications |
| [step4.md](step4.md) | End-to-end testing and troubleshooting |

Start with **step1.md** and work through them in order.

## Full reference

See [ServiceNow.md](../ServiceNow.md) at the repo root for all scripts and field definitions in one place.
