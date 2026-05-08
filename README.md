# Microsoft Sentinel Parsing — Team Tech Time (TTT)

## Goal of this session

By the end of this 3-hour TTT session, you will have:

1. **Understood why parsing matters** — Why custom log sources need normalization before they're useful in detections, and what the **Advanced Security Information Model (ASIM)** brings to the table
2. **Sent custom log data to Sentinel using a Logs Ingestion API** — Via Bruno, an app registration, a DCE, and a DCR (the modern, supported pipeline)
3. **Implemented ingest-time parsing** — Used a **DCR `transformKql`** to flatten and normalize the JSON *before* it lands in a custom table
4. **Implemented query-time parsing** — Wrote an **ASIM KQL parser function** that normalizes the same data *on read*, plugged it into the built-in unifying `_Im_Authentication` parser
5. **Run one detection rule against both** — Deployed an ASIM Authentication brute-force analytic rule that fires across both pipelines (and any other ASIM-compliant source) without modification

This session is **build-first, with explainers throughout** — every "do this" step includes a short *why* so you understand the concept, not just the clicks.

---

## The big picture

We are taking the **same fictional log source** (`ContosoAuth` sign-in events) and pushing it into Sentinel **two different ways**, so you can compare them side-by-side and decide which approach fits which use case.

```
                         Bruno (REST client)
                                │
                       Sample JSON payload
                                │
                  Microsoft Entra app registration
                  (Monitoring Metrics Publisher
                       on the DCRs)
                                │
                                ▼
                Data Collection Endpoint (DCE)
                                │
              ┌─────────────────┴──────────────────┐
              │                                    │
              ▼                                    ▼
   DCR-A  Ingest-time transform         DCR-B  Pass-through
   (transformKql flattens nested       (stores raw nested JSON
    JSON into ASIM column names)        as a dynamic column)
              │                                    │
              ▼                                    ▼
   Table:  ContosoAuthIngest_CL          Table:  ContosoAuthRaw_CL
   (already shaped like ASIM)            (raw — needs read-time parser)
              │                                    │
              │                          KQL function:
              │                          vimAuthenticationContosoAuth
              │                          (normalizes on read)
              │                                    │
              └─────────────────┬──────────────────┘
                                │
                                ▼
                ASIM unifying parser:  _Im_Authentication
                                │
                                ▼
              Analytic Rule:  Brute-force (ASIM)
              ── one rule, every Authentication source ──
```

Why two paths? Because Sentinel actually offers both, and the choice has real cost, latency, and flexibility implications. We will discuss the trade-offs after you've built each.

---

## Table of Contents

| Step | Title | Difficulty | Duration | What happens |
|------|-------|------------|----------|--------------|
| 1 | [Why Parsing & ASIM](Step1-Why-Parsing-and-ASIM.md) | Easy | ~25 min | Conceptual: the language analogy, the ASIM model, ingest-time vs query-time. **No clicks.** |
| 2 | [Prepare the Lab](Step2-Prepare-the-Lab.md) | Easy | ~30 min | App registration, DCE, Bruno, sample data. **One-time setup.** |
| 3 | [Ingest-time Parsing](Step3-Ingest-time-Parsing.md) | Medium | ~40 min | Custom table + DCR with `transformKql`. Send data. Verify. |
| 4 | [Query-time Parsing](Step4-Query-time-Parsing.md) | Medium | ~45 min | Custom table + passthrough DCR. Build the ASIM parser. Plug into `_Im_Authentication`. |
| 5 | [ASIM Detection Rule](Step5-ASIM-Detection-Rule.md) | Medium | ~25 min | Deploy the brute-force rule. Watch it fire on both paths. |
| 6 | [Closeout & Decision Matrix](Step6-Closeout.md) | — | ~15 min | Recap, when to choose what, cleanup. |

Total: **~3 hours**. Steps are sequential — each one builds on the previous.

---

## Prerequisites

### Access & licenses
- [ ] An **Azure subscription** with `Contributor` or higher on a resource group you can use
- [ ] A **Microsoft Sentinel-enabled Log Analytics workspace** (any region; ideally not production)
- [ ] Permission in **Microsoft Entra ID** to create an **app registration** and a **client secret**
- [ ] Permission to assign Azure RBAC (`User Access Administrator` or `Owner`) on the DCRs you create

### Tools
- [ ] **Bruno** — free, open-source REST client. Download: <https://www.usebruno.com/downloads>
- [ ] **A browser** — you will hop between the Azure portal, the Microsoft Sentinel portal, and Bruno

### Knowledge
- Comfortable reading **JSON** and basic **KQL**
- No prior ASIM, DCR, or Logs Ingestion API experience required — we will build it up

---

## How to use this guide

1. Follow the steps **in order** — Step 3 reuses the app reg and DCE you create in Step 2
2. Each step ends with a **"What you built"** checklist — verify before moving on
3. Lines marked with **💡 Why** explain the concept behind a click
4. Lines marked with **🔍 Explore** are optional deep-dives if you finish early

---

## Quick links (you'll come back to these)

- [ASIM Authentication schema reference](https://learn.microsoft.com/en-us/azure/sentinel/normalization-schema-authentication)
- [Develop a custom ASIM parser](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers)
- [Logs Ingestion API overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)
- [Tutorial: Send data to Logs (portal)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal)
- [Structure of a transformation in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations-kql)
- [ASIM Authentication-based analytic rules](https://learn.microsoft.com/en-us/azure/sentinel/normalization-content#authentication-security-content)

---

> **Ready?** Start with [Step 1 — Why Parsing & ASIM](Step1-Why-Parsing-and-ASIM.md) →
