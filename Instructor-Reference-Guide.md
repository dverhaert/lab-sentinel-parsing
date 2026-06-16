# Instructor Reference Guide

This short guide is for instructors running the Sentinel Parsing lab with learners.

## 1. Instructor prep checklist

- Confirm learners have access to a non-production workspace and resource group.
- Ask learners to complete Step 1 quickly before hands-on setup.
- Keep one known-good environment ready so you can demonstrate expected outputs.

## 2. Access rights (emphasize this first)

Most lab failures are permissions issues, not KQL issues.

Required rights:
- Azure subscription/resource group: Contributor (or higher) for resource creation.
- Microsoft Entra ID: permission to create app registrations and client secrets.
- RBAC assignment permission: Owner or User Access Administrator on the DCR scope.

Critical assignment for ingestion:
- Assign the app registration the `Monitoring Metrics Publisher` role on each DCR used in the lab.
- Do this for both paths:
  - Ingest-time DCR (Step 3)
  - Query-time/raw DCR (Step 4)

Quick troubleshooting cues:
- `401` when getting token: check `tenantId`, `clientId`, `clientSecret`.
- `403` on ingestion: usually missing `Monitoring Metrics Publisher` on the target DCR.
- No alerts later: verify both pipelines ingested data and parser mapping is active.

## 3. Timestamp instruction for lab data

Before running ingestion and detections, ensure sample event timestamps are:
- On today’s date (UTC)
- A few hours in the past (UTC)
- Never in the future

Recommended target window:
- 2 to 4 hours before current UTC time.

Why:
- Avoids delayed or missing detections caused by future timestamps.
- Keeps examples realistic for SOC triage and recent-query windows.

## 4. Suggested flow during delivery

1. Step 1: explain ingest-time vs query-time trade-offs with one example.
2. Step 2: enforce setup discipline (IDs, secrets, DCE URL, Bruno env values).
3. Step 3 and Step 4: repeatedly remind learners that DCR RBAC is mandatory.
4. Step 5: compare rule behavior across both pipelines.
5. Step 6: discuss decision matrix (cost, latency, flexibility, maintainability).

## 5. Instructor notes

- Keep language simple: parsing is schema normalization, not just field extraction.
- If learners get blocked, validate permissions first, then payload shape, then KQL.
- Encourage learners to validate each step before moving on.
