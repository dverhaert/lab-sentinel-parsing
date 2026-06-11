> **Tip:** If you used a unique suffix for your resources, remember to use it when searching for or deleting resources during cleanup.
# Step 6 — Closeout & Decision Matrix

[← Back: Step 5](Step5-ASIM-Detection-Rule.md) | [Back to overview](README.md)

---

## Table of Contents

- [Step 6 — Closeout \& Decision Matrix](#step-6--closeout--decision-matrix)
  - [Table of Contents](#table-of-contents)
  - [Recap](#recap)
  - [6.1 The decision matrix — when to use which](#61-the-decision-matrix--when-to-use-which)
  - [6.2 Cost \& performance notes](#62-cost--performance-notes)
  - [6.3 Common pitfalls](#63-common-pitfalls)
  - [6.4 Where to go next](#64-where-to-go-next)
    - [Reference reading](#reference-reading)
  - [6.5 Cleanup](#65-cleanup)
  - [Quick reference](#quick-reference)
    - [Ingest-time vs query-time in one line each](#ingest-time-vs-query-time-in-one-line-each)
    - [The five "must" rules of ASIM normalization](#the-five-must-rules-of-asim-normalization)
    - [The one rule that justifies the whole effort](#the-one-rule-that-justifies-the-whole-effort)
  - [Thank you \& feedback](#thank-you--feedback)

---

## Recap

In one session you:

1. Understood **why** parsing exists (different sources speak different languages) and how **ASIM** is the common dictionary
2. Stood up the modern, supported ingestion pipeline: **app reg → Logs Ingestion API → DCE → DCR → custom table**, driven from **Bruno**
3. Built **ingest-time parsing** with a DCR `transformKql` — flattening nested JSON to ASIM columns before storage
4. Built **query-time parsing** with an ASIM filtering parser function — keeping data raw and translating on read
5. Registered both pipelines with `Im_AuthenticationCustom`, so the built-in `_Im_Authentication` unifier exposes them
6. Ran a single brute-force detection rule that fired on **both** pipelines without any source-specific knowledge

The same 20 sample events now exist in your workspace twice, queryable through one ASIM surface, watched by one rule.

---

## 6.1 The decision matrix — when to use which

Use this in real onboarding decisions:

| Question | Answer suggests… |
|----------|------------------|
| Is the source schema **stable** (no surprise field renames)? | **Ingest-time** — bake it in; no need to keep raw |
| Is event volume **very high** (>>10M events/day)? | **Ingest-time** — query-time CPU adds up |
| Will many rules query this table **frequently**? | **Ingest-time** — pay parsing once, reap savings forever |
| Is the source **new / unfamiliar / experimental**? | **Query-time** — keep the raw record while you learn what matters |
| Do you sometimes need fields that **aren't in ASIM**? | **Query-time** (or hybrid: store raw, but also project key ASIM fields at ingest) |
| Is the parser likely to **need fixing** as you discover edge cases? | **Query-time** — fix the function, all historical data benefits |
| Are you under **strict storage budget** and can drop fields? | **Ingest-time** with aggressive `project` — drop everything you won't query |
| Do **compliance/forensics** require the original record verbatim? | **Query-time** (or hybrid: ingest-time normalized table + a low-retention raw archive) |

In practice, mature Sentinel deployments often run **both**: a richly normalized ingest-time table for fast detections, and a short-retention raw archive for forensics. Don't treat it as a binary choice.

---

## 6.2 Cost & performance notes

- **Ingest-time transform CPU is free** in the sense that there's no separate `transformKql` line item — but very expensive transforms can throttle ingestion. Keep them simple; defer complexity to query time.
- **Storage** is billed per GB ingested **after** the transform runs. So an ingest-time `project` that drops half the fields literally halves your bill on that source. Query-time parsing has no such lever — you store everything.
- **Query-time parsers** add CPU cost on every run. ASIM's filter-pushdown convention (the `*_has_any`, `eventtype_in`, etc. parameters) is what makes it tolerable. **Always honor that contract** when writing your own parsers.
- **`union isfuzzy=true`** in unifying parsers swallows errors silently. Periodically run each `vim*` directly to make sure none have quietly broken.
- **Scheduled rule frequency** matters: a rule running every 5 min querying `_Im_Authentication` over a 1h window means the same data is parsed ~12× per hour at query time. Multiply by the number of ASIM rules. This is the cost case for ingest-time on hot sources.

---

## 6.3 Common pitfalls

- **The `https://monitor.azure.com//.default` double slash.** Not a typo. Don't "fix" it.
- **Forgetting `TimeGenerated`** in the transform. The destination table requires it; ingest fails silently if the column doesn't exist or isn't a `datetime`.
- **403s on first POST.** RBAC propagation is 1–3 min. Wait, retry.
- **Editing the built-in `_Im_Authentication`** — you can't, and shouldn't try; use `Im_AuthenticationCustom` (which the built-in auto-invokes). Don't confuse it with the workspace-deployed `imAuthentication` (no underscore) which only exists if you deployed [aka.ms/DeployASIM](https://aka.ms/DeployASIM).
- **Not mapping `EventType` / `EventResult` to ASIM-allowed values.** Detections filter by ASIM vocabulary, not by raw vendor strings. `"signin"` ≠ `"Logon"`.
- **`project` in the DCR transform that drops a column you later wish you had.** It's gone. Re-ingestion is the only fix.
- **Writing a query-time parser without filter pushdown.** The parameters aren't decoration; rules that pass `eventresult="Failure"` expect you to *use* it before the heavy parsing.
- **Tokens expire after 1 hour.** Re-running `01 - Get Token` is faster than debugging a 401.
- **Case errors** Making small mistakes in lower/uppercase can lead to great frustration. Make sure to double check what variables you are changing, and please create a Pull Request to this repository if you see any mistakes.

---

## 6.4 Where to go next

- **Other ASIM schemas:** repeat this exercise for `NetworkSession`, `WebSession`, `ProcessEvent`, `Dns`, `FileEvent`. Same pattern, different schema.
- **ASIM parameter-less parsers:** add `ASimAuthenticationContosoAuth` as a thin wrapper on top of your `vim*` for ad-hoc hunting.
- **Hybrid pipelines:** one source, two DCRs — ingest-time to a normalized table for detections, passthrough to a cheap-tier raw table for retention.
- **Workbooks for ASIM:** the **`ASim Workbook`** in the Sentinel content hub visualizes which sources are wired up to each unifier.
- **Test your parsers:** the [ASIM parser test framework](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers#test-your-parsers) catches schema drift early.

### Reference reading

- [ASIM overview](https://learn.microsoft.com/en-us/azure/sentinel/normalization)
- [ASIM Authentication schema](https://learn.microsoft.com/en-us/azure/sentinel/normalization-schema-authentication)
- [Develop a custom ASIM parser](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers)
- [Logs Ingestion API overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)
- [Send data to Logs (portal tutorial)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal)
- [Data collection transformations in KQL](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations-kql)
- [ASIM-based analytic rules content](https://learn.microsoft.com/en-us/azure/sentinel/normalization-content)

---

## 6.5 Cleanup

If this was a personal lab subscription and you're done, delete to stop billing:

1. The **resource group** holding the DCE, both DCRs, and the custom tables — easiest is to put everything in a `rg-sentinel-parsing-lab-<yourinitials>` group up front and delete the whole group.
2. The **app registration** (e.g., `app-sentinel-parsing-lab-<yourinitials>`) (Microsoft Entra → App registrations → … → Delete).
3. The **KQL functions** you created (e.g., `vimAuthenticationContosoAuth`, `vimAuthenticationContosoAuthIngest`, `Im_AuthenticationContoso`) (workspace → Logs → Functions tab → delete).
4. The **analytic rule** you created in Step 5.

If this was a shared workspace, **leave the analytic rule disabled rather than delete it** — your colleagues may want to inspect it.

---

## Quick reference

### Ingest-time vs query-time in one line each

> **Ingest-time:** translator at the door. Fast queries, baked-in schema, no going back.  
> **Query-time:** translator at the microphone. Flexible, reversible, pays CPU on every query.

### The five "must" rules of ASIM normalization

1. **`TimeGenerated`** must exist in every destination table
2. **`EventType` / `EventResult`** must use ASIM-allowed values (not raw vendor strings)
3. **`EventVendor` / `EventProduct` / `EventSchema` / `EventSchemaVersion`** are mandatory
4. **Filtering parsers** must accept the standard parameters and apply pushdown
5. **Custom parsers** plug into the unifier via **`Im_<Schema>Custom`**, never by editing the built-in

### The one rule that justifies the whole effort

> One ASIM detection rule × N sources = N detections, for the cost of one.

---

🎉 **You're done. Thanks for showing up.**

---

## Thank you & feedback

This lab represents a deep dive into a real-world skillset that takes most engineers weeks to assemble on their own. **Thank you for investing your time here.**

If you found this guide useful, broken, or incomplete, **please reach out or open a pull request.** This is a living lab — your feedback and contributions make it better for everyone.

**How to contribute:**
- **Found a typo or error?** → Open a [Pull Request](https://github.com/your-repo-url) with a fix.
- **A step didn't work as written?** → File an issue or contact me; I want to know.
- **Want to add a new schema exercise** (NetworkSession, Dns, etc.)? → Reach out; I'll help you structure it.

Even if you just want to say "this was useful" or "this was confusing," that input shapes the next revision. No feedback is too small.

---

[← Back: Step 5](Step5-ASIM-Detection-Rule.md) | [Back to overview](README.md)
