# Step 1 — Why Parsing & ASIM

[← Back to overview](README.md) | [Next: Step 2 — Prepare the Lab →](Step2-Prepare-the-Lab.md)

---

## Table of Contents

- [Goal](#goal)
- [1.1 The problem: every log source speaks a different language](#11-the-problem-every-log-source-speaks-a-different-language)
- [1.2 The analogy: translators at the United Nations](#12-the-analogy-translators-at-the-united-nations)
- [1.3 What is ASIM?](#13-what-is-asim)
- [1.4 The two ways to parse: ingest-time vs query-time](#14-the-two-ways-to-parse-ingest-time-vs-query-time)
- [1.5 Why one detection rule beats fifty](#15-why-one-detection-rule-beats-fifty)
- [What you learned](#what-you-learned)

---

## Goal

Understand **why** Sentinel needs parsing, **what** ASIM solves, and **when** to use ingest-time versus query-time parsing — so the rest of the lab is "I know what I'm doing and why" instead of "I'm copy-pasting and hoping."

**Difficulty:** Easy  
**Duration:** ~25 minutes  
**Tools:** Just your brain and this page

---

## 1.1 The problem: every log source speaks a different language

Imagine you receive sign-in logs from **five different products**. Each one tells you the same story — *who logged in, from where, did it succeed* — but each one uses different words.

| Concept | Microsoft Entra ID | Okta | A custom Dutch HR app | A Linux server | Our `ContosoAuth` |
|---------|-------------------|------|-----------------------|----------------|-------------------|
| The user who logged in | `userPrincipalName` | `actor.alternateId` | `gebruikersnaam` | `user` | `event.user.upn` |
| The source IP | `ipAddress` | `client.ipAddress` | `bron_ip` | `rhost` | `event.src.ip` |
| Did it succeed? | `status.errorCode == 0` | `outcome.result == "SUCCESS"` | `gelukt == "ja"` | `"Accepted password"` in message | `event.outcome == "Success"` |

Now imagine you want to write **one** detection rule that says *"alert me when any user fails to log in 10 times in 5 minutes from the same IP."*

Without normalization, you'd have to:
- Write five different queries (one per product)
- Maintain them as schemas drift
- Re-do all of this when a sixth product is onboarded
- Hope nobody mistypes `gebruikersnaam`

This is the **parsing problem**: raw logs are *informational*, not *operational*. Detections need a single, consistent vocabulary.

---

## 1.2 The analogy: translators at the United Nations

Think of Sentinel as the **United Nations General Assembly hall**. Every country (log source) sends delegates speaking their own language: French, Mandarin, Dutch, Klingon. The Security Council (your analytic rules) needs to make decisions based on what's being said — but the Council members only speak **English**.

Two ways to solve this:

**Option A — Translate at the door (ingest-time):**  
A translator stands at the entrance. Before a delegate enters, they translate everything into English and hand the Council a clean transcript. Fast for the Council, but if the translator made a mistake, you can't go back to the original — and every translator slows the door down a bit.

**Option B — Translate at the microphone (query-time):**  
Delegates walk in speaking their own language. A live translator translates only when someone actually speaks. The original speech is preserved on tape forever; you can replay it and re-translate later if needed. But every Council session pays the translation cost in real time.

That's it. That's the whole TTT in one analogy:

- **Ingest-time parsing** = translator at the door = **DCR `transformKql`**
- **Query-time parsing** = translator at the microphone = **ASIM KQL parser function**
- **English** = **the ASIM schema**

> **💡 Why this matters:** every parsing decision in the rest of the lab is a re-application of this analogy. Keep it in your head.

---

## 1.3 What is ASIM?

The **Advanced Security Information Model (ASIM)** is Microsoft's open, schema-based normalization layer for Sentinel. It is *not* a product, *not* a license, *not* a connector. It is three things stacked together:

### 1. **Schemas** — the "English dictionary"
Standard column names and types for common security event categories. Examples:
- `Authentication` — sign-ins, sign-outs, password changes
- `NetworkSession` — flow records, firewall logs
- `WebSession` — proxy and WAF logs
- `ProcessEvent`, `FileEvent`, `RegistryEvent`, `Dns`, `Audit`, …

In every schema, fields like `EventVendor`, `EventProduct`, `EventResult`, `SrcIpAddr`, `TargetUserName` mean the same thing regardless of the original source.

📖 [ASIM Authentication schema reference](https://learn.microsoft.com/en-us/azure/sentinel/normalization-schema-authentication)

### 2. **Parsers** — the "translators"
Per-source KQL functions that convert raw data into the schema. Naming convention:
- `vimAuthenticationContosoAuth` — **filtering** parser (accepts time + filter parameters; faster, used in production by detection rules)
- `ASimAuthenticationContosoAuth` — **parameter-less** parser (a 1-line wrapper on top of the filtering one; used for ad-hoc hunting in the Logs editor)

You write one **pair** per source per schema. The pair shares logic — the parameter-less one just calls the filtering one with no filters.

> **💡 In this lab we build the filtering parser in the main flow** (Step 4.6) because that's what the Step 5 detection rule needs. The parameter-less wrapper is added in **Step 4.9 (Bonus)** — it takes ~5 minutes once the filtering one works.

### 3. **Unifying parsers** — the "Council clerk who reads everyone's transcript"
Built-in functions that **union all parsers for a schema** into one queryable surface.

```
_Im_Authentication          ← filtering, used by detections (built-in, ships in every workspace)
_ASim_Authentication        ← parameter-less, used for hunting
```

> **💡 Note on naming:** Microsoft also offers a *workspace-deployed* version of these unifiers (`imAuthentication` / `ASimAuthentication`, no leading underscore) that you can install via the [aka.ms/DeployASIM](https://aka.ms/DeployASIM) ARM template. They're functionally equivalent. **In this lab we use the built-in `_Im_Authentication`** because it's already there — no extra deployment step. If you ever see a `Failed to resolve table or column expression named 'imAuthentication'` error, you've hit the difference.

When you query `_Im_Authentication`, Sentinel transparently calls `vimAuthenticationAAD`, `vimAuthenticationOkta`, `vimAuthenticationContosoAuth`, etc., and returns the union — all already in ASIM shape.

> **💡 Why it works:** the unifying parser is *also* a KQL function. You can extend it (add your own custom source) by registering your parser through the **`Im_AuthenticationCustom`** custom unifying parser — which the built-in unifier automatically calls if it exists. We will create that in Step 4.

📖 [Develop a custom ASIM parser](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers)

---

## 1.4 The two ways to parse: ingest-time vs query-time

|  | **Ingest-time (DCR `transformKql`)** | **Query-time (ASIM parser function)** |
|---|---|---|
| **When parsing happens** | Once, during ingestion, before data lands in the table | Every time a query runs |
| **What's stored** | Already-normalized columns | The original raw record |
| **Typical table shape** | Looks like ASIM (`SrcIpAddr`, `TargetUserName`, …) | Whatever the source sent |
| **Storage cost** | Pay for normalized columns only | Pay for raw data; can be cheaper if you drop fields |
| **Compute cost** | Paid once, on ingest (small per-event) | Paid every query (can add up on large datasets) |
| **Reversibility** | If the transform was wrong, the raw data is **lost** | Original data is intact; fix the parser, rerun |
| **Who can change it** | DCR change → redeploy → only future events affected | Edit the function → all past data is "re-parsed" instantly |
| **Best when** | Schema is stable, volume is high, queries are frequent | Schema is fluid, you might need raw fields later, exploration phase |

> **💡 Why we will do both today:** so you can feel the difference. The ingest-time table will be lean and pre-shaped. The query-time table will be raw and "messy", and you will see your parser function clean it up live.

📖 [Data collection transformations in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations)

---

## 1.5 Why one detection rule beats fifty

Here is the punchline of the entire session.

You have ten sign-in sources. You want to detect **brute-force attacks** (many failed sign-ins for one user from one IP within a short window).

**Without ASIM:**
- 10 detection rules, one per table
- 10 places to fix when threshold logic changes
- Onboarding source 11 = writing rule 11

**With ASIM:**
- 1 rule, queries `_Im_Authentication`
- Filter logic written once: `where EventResult == "Failure"` and `summarize count() by TargetUserName, SrcIpAddr`
- Onboarding source 11 = write a parser, register it in `Im_AuthenticationCustom`. **Zero change to the rule.**

This is exactly what the built-in **"Brute force attack against user credentials (Uses Authentication Normalization)"** rule does — and in Step 5 we will deploy it (or a simplified version) and watch it fire on `ContosoAuth` data **without ever telling it about ContosoAuth**.

📖 [ASIM Authentication security content](https://learn.microsoft.com/en-us/azure/sentinel/normalization-content#authentication-security-content)

---

## What you learned

- [ ] Parsing exists because every log source uses different field names and values for the same concepts
- [ ] ASIM = **schemas** (vocabulary) + **parsers** (translators) + **unifying parsers** (one query surface per schema)
- [ ] **Ingest-time** parsing is a DCR `transformKql` — fast at query time, but the raw form is gone
- [ ] **Query-time** parsing is a KQL function — flexible and reversible, but pays compute on every query
- [ ] One ASIM detection rule covers every source whose parser is registered with the unifying parser

---

[← Back to overview](README.md) | [Next: Step 2 — Prepare the Lab →](Step2-Prepare-the-Lab.md)
