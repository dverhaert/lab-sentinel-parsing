> **Tip:** When creating KQL functions or analytic rules, add a unique suffix (e.g., `vimAuthenticationCustomAuthIngest_dv` to avoid conflicts in shared workspaces.
# Step 5 — One ASIM Detection Rule, Both Sources

[← Back: Step 4](Step4-Query-time-Parsing.md) | [Next: Step 6 — Closeout →](Step6-Closeout.md)

---

## Table of Contents

- [Step 5 — One ASIM Detection Rule, Both Sources](#step-5--one-asim-detection-rule-both-sources)
  - [Table of Contents](#table-of-contents)
  - [Goal](#goal)
  - [5.1 Wire the ingest-time table into ASIM too](#51-wire-the-ingest-time-table-into-asim-too)
    - [Create `vimAuthenticationContosoAuthIngest`](#create-vimauthenticationcontosoauthingest)
    - [Update `Im_AuthenticationCustom` to include both](#update-im_authenticationcustom-to-include-both)
  - [5.2 The rule we're going to deploy](#52-the-rule-were-going-to-deploy)
  - [5.3 Option A — Deploy the built-in template](#53-option-a--deploy-the-built-in-template)
  - [5.4 Option B — Create a simplified rule from scratch](#54-option-b--create-a-simplified-rule-from-scratch)
  - [5.5 Trigger and observe the alert](#55-trigger-and-observe-the-alert)
    - [Quick option — preview the rule logic now](#quick-option--preview-the-rule-logic-now)
    - [Real-time option — re-send the data with current timestamps](#real-time-option--re-send-the-data-with-current-timestamps)
  - [5.6 The "aha" moment](#56-the-aha-moment)
  - [What you built](#what-you-built)

---

## Goal

Deploy a single ASIM-based brute-force analytic rule and watch it fire on **both** of our pipelines (and would fire on any other ASIM-compliant Authentication source) **without any source-specific logic in the rule**.

**Difficulty:** Medium  
**Duration:** ~25 minutes  
**Tools:** Microsoft Sentinel, KQL editor

---

## 5.1 Wire the ingest-time table into ASIM too

Step 4 plugged `ContosoAuthRaw_CL` (via `vimAuthenticationContosoAuth`) into the built-in `_Im_Authentication` unifier (through `Im_AuthenticationCustom`). The ingest-time table `ContosoAuthIngest_CL` from Step 3 is **already in ASIM shape**, but it's *not yet visible* through `_Im_Authentication` — we never registered it.

We'll add a **trivial wrapper parser** so that table also flows through the unifier. This gives us the perfect side-by-side demo: one detection rule, two custom tables, both detected.

### Create `vimAuthenticationContosoAuthIngest`

In the **Logs** editor, paste the body below and **Save as function** with name `vimAuthenticationContosoAuthIngest`. Add the **same seven parameters** in the Save dialog as you did for `vimAuthenticationContosoAuth` in Step 4.6 (single-field variant string is identical — copy from Step 4.6 if needed).

```kusto
ContosoAuthIngest_CL
| where not(disabled)
| where (isnull(starttime) or TimeGenerated >= starttime)
    and (isnull(endtime)   or TimeGenerated <= endtime)
| where (array_length(targetusername_has_any) == 0
         or TargetUsername has_any (targetusername_has_any))
    and (array_length(srcipaddr_has_any_prefix) == 0
         or has_any_ipv4_prefix(SrcIpAddr, srcipaddr_has_any_prefix))
    and (array_length(eventtype_in) == 0
         or EventType in (eventtype_in))
    and (eventresult == "*" or EventResult == eventresult)
| extend
    User           = TargetUsername,
    IpAddr         = SrcIpAddr,
    Dvc            = SrcDvcHostname,
    SrcDvcIpAddr   = column_ifexists("SrcDvcIpAddr", ""),
    TargetUserType = column_ifexists("TargetUserType", "")
```

> **💡 Why add these two fields:** they keep compatibility with ASIM rules/template variants that expect `SrcDvcIpAddr` and `TargetUserType`, while safely falling back to this table's current columns.

Same parameters as `vimAuthenticationContosoAuth`, **dramatically less code** — because the table already has ASIM column names. No `RawEvent.user.upn` digging required.

> **💡 Why this is so much shorter:** this is the ingest-time payoff in plain sight. The transform did the hard work once at ingest; the parser becomes a thin filter wrapper. Compare the two parser bodies side by side and you can *see* the trade-off we discussed in Step 1.

### Update `Im_AuthenticationCustom` to include both

Edit `Im_AuthenticationCustom` (Functions tab → right-click → **Load function code into editor**), replace the body, then **Save → Save as function** (overwrite). **Keep the same 12 parameters** you defined in Step 4.8 — the unifier contract didn't change, only the body.

Body:

```kusto
let DisabledParsers = materialize(
    _GetWatchlist('ASimDisabledParsers')
    | where SearchKey in ('Any',
                          'ExcludevimAuthenticationContosoAuth',
                          'ExcludevimAuthenticationContosoAuthIngest')
    | extend SourceSpecificParser = column_ifexists('SourceSpecificParser', '')
    | distinct SourceSpecificParser
);
union isfuzzy=true
    vimAuthenticationContosoAuth(
        starttime, endtime, targetusername_has_any, srcipaddr_has_any_prefix,
        eventtype_in, eventresult,
        ('ExcludevimAuthenticationContosoAuth' in (DisabledParsers))),
    vimAuthenticationContosoAuthIngest(
        starttime, endtime, targetusername_has_any, srcipaddr_has_any_prefix,
        eventtype_in, eventresult,
        ('ExcludevimAuthenticationContosoAuthIngest' in (DisabledParsers)))
```

> **💡 Why we don't add new parameters when adding a second source:** the 12-parameter signature is between `Im_AuthenticationCustom` and the **built-in unifier `_Im_Authentication`**. The fact that we now call two inner parsers is invisible to that contract. Adding inner parsers never changes the outer signature.

**Save**. Then verify both pipelines show up in the unifier:

You should see **`AuthGateway`** with **40 events** total (20 from each table). Same logical events, two storage strategies, one query interface. *That's ASIM working.*

```kusto
_Im_Authentication
| where EventVendor == "ContosoAuth"
| summarize Events = count() by EventProduct
```

---

## 5.2 The rule we're going to deploy

The detection logic for brute force on the ASIM Authentication schema is:

> Alert when the **same `TargetUsername`** has **≥ 5 failed authentications** from the **same `SrcIpAddr`** within a **5-minute** window.

Notice what's **not** in that sentence:
- No mention of which table the data lives in
- No mention of `ContosoAuth`, `event.outcome`, `event.user.upn`
- No mention of nested JSON or transformations

The rule operates entirely on **ASIM vocabulary**, and `_Im_Authentication` does the rest.

You have two ways to ship this rule. Pick one.

---

## 5.3 Option A — Deploy the built-in template

Microsoft Sentinel ships a maintained brute-force template that uses ASIM normalization.

1. **Microsoft Sentinel** portal → your workspace → **Content hub**
2. Search for **"ASIM"** or **"Authentication"** and install the **`ASim Authentication Domain Solution`** (or the equivalent content pack — the exact name evolves; look for one whose description mentions "Uses Authentication Normalization")
3. After install, go to **Analytics** → **Rule templates**
4. Find **"Brute force attack against user credentials (Uses Authentication Normalization)"**  
   — and/or — **"Potential Password Spray Attack (Uses Authentication Normalization)"**
5. Click **Create rule** → step through the wizard:
   - **Severity:** Medium
   - **Run frequency:** every 5 minutes (or as low as you can — for the demo we want it to fire fast)
   - **Lookup data from the last:** 1 hour
   - **Stop running query after:** 1 hour
    - **Entity mapping:** this is not always auto-populated. If it is empty, map it manually: `Account → TargetUsername` and `IP → IpAddresses`.
6. **Review** → **Create**

> **🛠️ Troubleshooting template query resolution in this lab:** some built-in template variants reference `imAuthentication` (lowercase `i`) plus the ASIM columns `SrcDvcIpAddr` and `TargetUserType`. In a lab workspace, make it resolve by either (a) editing the template query to use `_Im_Authentication`, or (b) creating an alias function named `imAuthentication` with body `_Im_Authentication` (that's it, no parameters necessary). Also ensure both custom `vim*` parsers emit `SrcDvcIpAddr` and `TargetUserType` (see [5.1](#51-wire-the-ingest-time-table-into-asim-too) and [Step 4.6](Step4-Query-time-Parsing.md#46-build-the-asim-filtering-parser-vimauthenticationcontosoauth)).

> **💡 Why use the built-in:** it's curated, threshold-tuned, has proper entity mapping, suppression logic, and gets updated by Microsoft. In production, prefer this 100% of the time.

📖 [ASIM Authentication security content](https://learn.microsoft.com/en-us/azure/sentinel/normalization-content#authentication-security-content)

---

## 5.4 Option B — Create a simplified rule from scratch

If the content pack isn't available in your tenant or you want to *see* the rule logic, build it yourself.

1. Microsoft Sentinel → **Analytics** → **+ Create** → **Scheduled query rule**
2. **General:**
   - **Name:** `TTT - ASIM Brute Force (custom)`
   - **Severity:** Medium
   - **Tactics:** Credential Access
3. **Set rule logic** — paste this query:

   ```kusto
   let lookback = 1h;
   let window   = 5m;
   let threshold = 5;
   _Im_Authentication
   | where TimeGenerated > ago(lookback)
   | where EventType == "Logon" and EventResult == "Failure"
   | summarize
       FailedAttempts = count(),
       StartTime = min(TimeGenerated),
       EndTime   = max(TimeGenerated),
       Vendors   = make_set(EventVendor),
       Products  = make_set(EventProduct)
       by TargetUsername, SrcIpAddr, bin(TimeGenerated, window)
   | where FailedAttempts >= threshold
   | extend
       AccountCustomEntity = TargetUsername,
       IPCustomEntity      = SrcIpAddr
   ```

4. **Entity mapping:**
   - `Account` → identifier `Name`, column `TargetUsername`
   - `IP` → identifier `Address`, column `SrcIpAddr`
5. **Query scheduling:** every 5 minutes, lookup last 1 hour
6. **Alert threshold:** Trigger when number of query results is **greater than 0**
7. **Review** → **Create**

Let's read the query out loud — **the only thing it knows is ASIM**:

| Clause | What it depends on |
|--------|-------------------|
| `_Im_Authentication` | The built-in unifying parser. **No table names.** |
| `EventType == "Logon"` | ASIM vocabulary value. Same on every source. |
| `EventResult == "Failure"` | ASIM vocabulary value. Same on every source. |
| `summarize ... by TargetUsername, SrcIpAddr` | ASIM column names. Same on every source. |

**No reference to `ContosoAuth`, no `RawEvent.user.upn`, no `event.outcome`.** Onboard a tenth source tomorrow with its own `vim*` parser, and this rule starts watching it for free.

---

## 5.5 Trigger and observe the alert

Our 20 sample events span only ~45 minutes of timestamps in early May 2026, so the rule's "last 1 hour" lookback will **not** match by the time you finish the lab. Two options to force a hit:

### Quick option — preview the rule logic now

Run the rule's query manually with a wide lookback:

```kusto
let lookback = 365d;     // wide enough to catch the sample data regardless of when you run
let window   = 5m;
let threshold = 5;
_Im_Authentication
| where TimeGenerated > ago(lookback)
| where EventType == "Logon" and EventResult == "Failure"
| summarize
    FailedAttempts = count(),
    Vendors   = make_set(EventVendor),
    Products  = make_set(EventProduct),
    Sources   = make_set(Type)        // _CL table the row came from
    by TargetUsername, SrcIpAddr, bin(TimeGenerated, window)
| where FailedAttempts >= threshold
```

Expected output: **two rows** for `beth@contoso.nl` from `185.220.101.42`, with `FailedAttempts >= 9`. One row's `Sources` set will contain `ContosoAuthIngest_CL`, the other's will contain `ContosoAuthRaw_CL`.

> **💡 Why two rows and not one:** each table contributes its own copy of the burst. They're literally the same logical events, ingested twice. **The rule didn't have to know that.** It just trusted ASIM.

### Real-time option — re-send the data with current timestamps

If you want the **scheduled** rule to actually fire and create an incident, edit `auth-events.json` to use timestamps within the last 30 minutes (or just bulk-replace the date), re-send via Bruno requests `02` and `03`, and wait for the next 5-minute scheduled run.

You can also run this as a side-by-side demo with **both** sample files (`auth-events.json` and `auth-events-channel2.json`): send each file through Bruno requests `02` and `03`, then watch `_Im_Authentication` and Incidents to see detections coming from the two different sources/payloads through the same ASIM rule.

---

## 5.6 The "aha" moment

Look at the alert (or the preview query result) and ask yourself:

- ✅ Did the rule know our table existed?
- ✅ Did the rule know whether parsing happened at ingest or query time?
- ✅ Did the rule know any vendor-specific field name?

**No to all three.**

That is the entire point of ASIM. **The detection author writes against the schema; the platform engineer wires up the sources.** Two roles, two skill sets, decoupled work.

> **🔍 Explore — the multiplier effect:** suppose you onboard Okta tomorrow. You write `vimAuthenticationOkta`, register it in `Im_AuthenticationCustom`, and walk away. This brute-force rule starts watching Okta. The "Potential Password Spray" rule starts watching Okta. Any future ASIM Authentication rule starts watching Okta. **That's the leverage.**

---

## What you built

- [ ] `vimAuthenticationContosoAuthIngest` — trivial wrapper for the already-flat ingest-time table
- [ ] Updated `Im_AuthenticationCustom` to union both `vim*` parsers
- [ ] An analytic rule (built-in template **or** custom) querying `_Im_Authentication`
- [ ] Entity mapping configured (`Account` / `IP`)
- [ ] Verified the rule logic detects the brute-force burst in **both** pipelines
- [ ] Internalized that the rule has zero source-specific logic — that's the entire point

---

[← Back: Step 4](Step4-Query-time-Parsing.md) | [Next: Step 6 — Closeout →](Step6-Closeout.md)
