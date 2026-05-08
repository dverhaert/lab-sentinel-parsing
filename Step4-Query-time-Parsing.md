# Step 4 — Query-time Parsing

[← Back: Step 3](Step3-Ingest-time-Parsing.md) | [Next: Step 5 — ASIM Detection Rule →](Step5-ASIM-Detection-Rule.md)

---

## Table of Contents

- [Goal](#goal)
- [4.1 What we're building (and why)](#41-what-were-building-and-why)
- [4.2 Create a passthrough custom table + DCR](#42-create-a-passthrough-custom-table--dcr)
- [4.3 Grant the app permission and capture IDs](#43-grant-the-app-permission-and-capture-ids)
- [4.4 Send the data via Bruno](#44-send-the-data-via-bruno)
- [4.5 Verify the raw data is there](#45-verify-the-raw-data-is-there)
- [4.6 Build the ASIM filtering parser `vimAuthenticationContosoAuth`](#46-build-the-asim-filtering-parser-vimauthenticationcontosoauth)
- [4.6.5 What the parameters actually *do* — buffet vs. menu](#465-what-the-parameters-actually-do--buffet-vs-menu)
- [4.7 Test the parser](#47-test-the-parser)
- [4.8 Register the parser with the unifying parser `_Im_Authentication`](#48-register-the-parser-with-the-unifying-parser-_im_authentication)
- [4.9 Bonus — complete the pair with `ASimAuthenticationContosoAuth`](#49-bonus--complete-the-pair-with-asimauthenticationcontosoauth)
- [What you built](#what-you-built)

---

## Goal

Build the **second** pipeline: the **query-time** path. Data is stored **as-is** in nested form, and a KQL parser function reshapes it into ASIM **on every query**. We then plug that parser into the built-in unifying `_Im_Authentication` parser so detection rules pick it up automatically.

**Difficulty:** Medium  
**Duration:** ~45 minutes  
**Tools:** Azure portal, Bruno, KQL editor

---

## 4.1 What we're building (and why)

```
Bruno  ──►  DCE  ──►  DCR (no transform)  ──►  ContosoAuthRaw_CL (nested JSON preserved)
                                                             │
                                                             ▼
                                            vimAuthenticationContosoAuth   ◄── KQL function (parses on read)
                                                             │
                                                             ▼
                                            Im_AuthenticationCustom        ◄── you register your parser here
                                                             │
                                                             ▼
                                                 _Im_Authentication         ◀── built-in unifying parser
                                                             │
                                                             ▼
                                            (any ASIM Authentication query/rule)
```

> **💡 Why a parameter-rich filtering function:** ASIM filtering parsers (`vim*`) accept time and field-level filter parameters and push those filters down **before** parsing — so a query that asks for "failed sign-ins from one IP in the last hour" doesn't waste CPU parsing 10 million success events. This is the single most important performance technique in ASIM.

📖 [Develop a custom ASIM parser](https://learn.microsoft.com/en-us/azure/sentinel/normalization-develop-parsers)

---

## 4.2 Create a passthrough custom table + DCR

Same wizard as Step 3, with two key differences: a **different table name**, and a **trivial transform** that keeps the data nested.

1. Workspace → **Tables** → **+ Create** → **New custom log (DCR-based)**
2. **Basics:**
   - **Table name:** `ContosoAuthRaw` → final name `ContosoAuthRaw_CL`
   - **DCR:** **Create new** → `dcr-contosoauth-raw`
   - **DCE:** the same `dce-sentinel-parsing-ttt` from Step 2
3. **Schema and transformation:** upload `Parsing/sample-data/auth-events.json` (same file as Step 3)
4. In the **Transformation editor**, replace whatever the portal generated with:

   ```kusto
   source
   | extend
       TimeGenerated = todatetime(timestamp),
       RawEvent      = event
   | project TimeGenerated, RawEvent
   ```

   That's it. Two columns: `TimeGenerated` (mandatory) and `RawEvent` (the entire nested blob, dynamic).

5. **Apply** → **Next** → **Create**.

> **💡 Why we still use a transform even though we're "not parsing":** the destination table **must** have a `TimeGenerated` column, but our payload calls it `timestamp`. So the transform's only real job is: rename `timestamp` → `TimeGenerated`, and keep everything else as one dynamic blob (`RawEvent`). This is the simplest possible transform — sometimes called a "passthrough."

> **💡 Why store everything in `RawEvent` and not as separate columns:** because we don't want to commit to a schema today. Tomorrow the source could add `event.app.deviceTrust` — by keeping the blob, our parser function can pick that up the moment we update it, with zero pipeline change.

---

## 4.3 Grant the app permission and capture IDs

Same drill as Step 3.4 / 3.5, but for the **new** DCR.

1. DCR `dcr-contosoauth-raw` → **Access control (IAM)** → assign **`Monitoring Metrics Publisher`** to `app-sentinel-parsing-ttt`
2. **JSON view** → copy `properties.immutableId` → save as Bruno env var `dcrImmutableIdRaw`
3. The stream name will be `Custom-ContosoAuthRaw_CL` → save as Bruno env var `streamRaw`

---

## 4.4 Send the data via Bruno

1. Duplicate your `02 - Send to Ingest DCR` request → rename to `03 - Send to Raw DCR`
2. Change the URL to use the **raw** variables:

   ```
   {{dceEndpoint}}/dataCollectionRules/{{dcrImmutableIdRaw}}/streams/{{streamRaw}}?api-version=2023-01-01
   ```

3. Body stays identical — same JSON file as Step 3.
4. (If you've been on the lab > 1 hour, re-run `01 - Get Token` first — tokens are valid for 1 hour.)
5. **Send** → **204 No Content**.

---

## 4.5 Verify the raw data is there

```kusto
ContosoAuthRaw_CL
| take 5
```

You should see the `RawEvent` column with **dynamic JSON values** that you can expand inline. Notice how unfriendly this is for a detection author — every field has to be addressed as `RawEvent.user.upn`, every value is `tostring()`-cast, and there's no consistent vocabulary with other Sentinel tables.

That's the problem the parser solves.

---

## 4.6 Build the ASIM filtering parser `vimAuthenticationContosoAuth`

We're going to save a KQL **function** — a saved query you can call by name with parameters.

1. Open the workspace → **Logs**
2. Paste the query below into the editor (don't run it yet — we will save it as a function).

   **Important:** the parameters are *not* defined in the body. They are defined separately in the **Save as function** dialog (next step). The body just *uses* them.

```kusto
ContosoAuthRaw_CL
| where not(disabled)
// ── push down time filter BEFORE parsing the dynamic blob (perf win) ──
| where (isnull(starttime) or TimeGenerated >= starttime)
    and (isnull(endtime)   or TimeGenerated <= endtime)
// ── extract just enough to apply the cheap filters ──
| extend
    _upn      = tostring(RawEvent.user.upn),
    _srcip    = tostring(RawEvent.src.ip),
    _rawType  = tostring(RawEvent.type),
    _outcome  = tostring(RawEvent.outcome)
| extend EventType = case(
    _rawType == "signin",          "Logon",
    _rawType == "signout",         "Logoff",
    _rawType == "password_change", "Other",
    "Other")
| extend EventResult = case(
    _outcome == "Success", "Success",
    _outcome == "Failure", "Failure",
    "Other")
// ── push down the rest of the cheap filters ──
| where (array_length(targetusername_has_any) == 0
         or _upn has_any (targetusername_has_any))
    and (array_length(srcipaddr_has_any_prefix) == 0
         or has_any_ipv4_prefix(_srcip, srcipaddr_has_any_prefix))
    and (array_length(eventtype_in) == 0
         or EventType in (eventtype_in))
    and (eventresult == "*" or EventResult == eventresult)
// ── now do the expensive full normalization, only on rows that survived ──
| extend
    EventVendor        = tostring(RawEvent.vendor),
    EventProduct       = tostring(RawEvent.product),
    EventSchema        = "Authentication",
    EventSchemaVersion = "0.1.4",
    EventCount         = int(1),
    EventStartTime     = TimeGenerated,
    EventEndTime       = TimeGenerated,
    EventOriginalUid   = tostring(RawEvent.id),
    EventOriginalType  = _rawType,
    EventResultDetails = tostring(RawEvent.reason),
    TargetUserName     = _upn,
    TargetUsernameType = "UPN",
    TargetUserId       = tostring(RawEvent.user.id),
    SrcIpAddr          = _srcip,
    SrcPortNumber      = toint(RawEvent.src.port),
    SrcGeoCountry      = tostring(RawEvent.src.geo.country),
    SrcGeoCity         = tostring(RawEvent.src.geo.city),
    SrcDvcHostname     = tostring(RawEvent.device.hostname),
    SrcDvcOs           = tostring(RawEvent.device.os),
    TargetAppName      = tostring(RawEvent.app.name),
    TargetSessionId    = tostring(RawEvent.app.sessionId),
    // ── ASIM aliases that detection rules may use ──
    User               = _upn,
    IpAddr             = _srcip,
    Dvc                = tostring(RawEvent.device.hostname)
| project-away _upn, _srcip, _rawType, _outcome
```

> **⚠️ If you try to *run* the query as-is, you'll get errors about undefined names** (`disabled`, `starttime`, …). That is **expected** — those names only exist once the editor knows they're function parameters. Skip running it; go straight to saving.

3. Click **Save** (top of the query editor) → **Save as function**
4. Fill in:
   - **Function name:** `vimAuthenticationContosoAuth`
   - **Legacy category:** `ASIM`
   - **Parameters:** click **+ Add parameter** and add each row below. (Some portal versions show a single text field — in that case paste the comma-separated string further below.)

     | Name | Type | Default value |
     |------|------|---------------|
     | `starttime` | `datetime` | `datetime(null)` |
     | `endtime` | `datetime` | `datetime(null)` |
     | `targetusername_has_any` | `dynamic` | `dynamic([])` |
     | `srcipaddr_has_any_prefix` | `dynamic` | `dynamic([])` |
     | `eventtype_in` | `dynamic` | `dynamic([])` |
     | `eventresult` | `string` | `"*"` |
     | `disabled` | `bool` | `false` |

     If you see a single text box, paste this exact string:

     ```
     starttime:datetime=datetime(null), endtime:datetime=datetime(null), targetusername_has_any:dynamic=dynamic([]), srcipaddr_has_any_prefix:dynamic=dynamic([]), eventtype_in:dynamic=dynamic([]), eventresult:string="*", disabled:bool=false
     ```

5. **Save**.

> **💡 Why this is the fix for your error:** the message `function expects 0 argument(s)` means the saved function has **zero parameters defined at the function level**. A `let parser = (...) { ... }; parser(...)` block inside the body is just an internal lambda — to the workspace, the function's public signature is whatever you typed (or didn't) in the **Parameters** UI of the Save dialog. Defining the parameters there is what makes `vimAuthenticationContosoAuth(eventresult="Failure")` work.
>
> If you already saved the function with no parameters: open it again (Logs → Functions tab → right-click `vimAuthenticationContosoAuth` → **Edit function details**), add the seven parameters as above, **Save**. No need to change the body.

Now let's unpack **why** this looks the way it does — every section is intentional:

| Block | Why |
|-------|-----|
| The seven parameters declared in the Save dialog | This is the **standard ASIM filtering parser signature**. The unifying `_Im_Authentication` parser (via `Im_AuthenticationCustom`) will call us with these exact parameter names. Defaults (`datetime(null)`, empty arrays, `"*"`) mean "no filter — return everything." |
| `where not(disabled)` | The `disabled` parameter is the kill-switch. If a detection rule passes `disabled=true`, the parser short-circuits and returns nothing. ASIM uses this for emergency disabling without breaking dependent queries. |
| Time filter applied **before** parsing | The whole point of a filtering parser. `TimeGenerated` is the table's indexed column — filtering on it first means the engine never reads rows outside the window. |
| `_upn` / `_srcip` / `_rawType` / `_outcome` extracted **early** | We need them for filter pushdown. We extract them once (cheap), then filter, then do the heavy normalization only on survivors. |
| `EventType` / `EventResult` mapped **before** filter | Detection rules filter by ASIM values (`"Logon"`, `"Failure"`), not raw vendor values. So we map first, then filter. |
| Filter pushdown for `targetusername_has_any`, `srcipaddr_has_any_prefix`, `eventtype_in`, `eventresult` | Each one is wrapped in `array_length(...) == 0 or ...` so an empty array means "don't filter on this." This is the ASIM convention — a rule that doesn't care about, say, source IP just doesn't pass `srcipaddr_has_any_prefix`. |
| The "expensive" normalization at the end | All those `extend` calls into the dynamic blob would be wasteful on filtered-out rows. Putting them after the filter pushdown is the perf optimization. |
| `User`, `IpAddr`, `Dvc` aliases | ASIM defines short aliases for the common entity columns. Some rules use them; mapping them is two cheap lines. |
| The final `parser(starttime, endtime, ...)` invocation | Required so saving as a function actually runs the lambda with the function's own parameters. |

> **💡 Why the `vim` prefix and not `ASim`:** convention. `vim*` = filtering (with parameters), `ASim*` = parameter-less convenience wrapper. Detection rules call `vim*` for performance. Hunters typing ad-hoc queries call `ASim*` because it's shorter. We're building the high-perf one; the parameter-less one is a 2-line wrapper you can add later if you want.

---

## 4.6.5 What the parameters actually *do* — buffet vs. menu

Before we test the parser, take five minutes to internalize what the parameters are *for*. Once it clicks, every ASIM parser you ever read makes sense.

### Two words you need to know first

The rest of this section uses two terms a lot. Neither is scary once you see what they mean.

**`parse_json` = "open the sealed envelope."**  
Our `ContosoAuthRaw_CL` table has a column called `RawEvent`. Each cell in that column holds one **sealed envelope** of nested JSON like:

```json
{ "user": {"upn":"beth@contoso.nl"}, "src": {"ip":"185.220.101.42"}, "outcome":"Failure", … }
```

Anytime our parser writes `tostring(RawEvent.user.upn)`, KQL has to **open that envelope and read inside it**. That's `parse_json` — the small but real cost of cracking open a JSON blob, **per row**. With 20 rows you don't notice. With 20 million rows you very much do.

What's *cheap*, by contrast, is reading **columns that are already columns** — like `TimeGenerated`. Those values sit on disk in a sorted, indexed format. KQL can skip whole blocks of them without ever opening an envelope.

> **One-line rule:** filtering on `TimeGenerated` is cheap. Filtering on anything *inside* `RawEvent` requires opening envelopes — so we want to do it on as few envelopes as possible.

**"Optimizer" = the smart waiter.**  
When you submit a KQL query, Kusto doesn't run it line-by-line in the order you typed. A planner — the **optimizer** — looks at the *whole* query and rearranges the work to do the cheap, narrowing steps **first**. Same idea as a smart waiter who reads your whole order before walking into the kitchen, and groups the work to save trips.

Two facts about the optimizer matter for us:

1. **It can see inside saved functions.** When you call `vimAuthenticationContosoAuth(…)`, Kusto doesn't treat the function as a black box. It pastes the body into your query and optimizes the **combined** thing. (This is called "inlining" — the recipe is copied straight onto the chef's prep sheet rather than being mailed in a sealed letter.)
2. **It can only push filters down past steps it understands.** A `where` on `TimeGenerated` can be pushed all the way down to disk. A `where` on a column that doesn't exist yet — because it gets created later by `parse_json`/`extend` — *cannot* be pushed past the step that creates it. **Order in the parser body matters enormously.**

That's all the theory. Now the analogy.

### The analogy: buffet vs. menu

We loaded **20 events** into `ContosoAuthRaw_CL`. Imagine that's actually **20 million** in production. A detection rule runs **every 5 minutes** and looks at the **last hour** for failed sign-ins from suspicious IPs.

**Without parameters → the buffet (`ASimAuthenticationContosoAuth`)**
> The kitchen prepares *every* dish on the menu, plates each one, and lays the entire spread out on the table. *Then* you walk down the line and pick the three plates you wanted.  
> Sentinel reads all 20M envelopes, opens each one (`parse_json`), builds the full ASIM column set, and *then* the rule's `where TimeGenerated > ago(1h) | where EventResult == "Failure"` throws 19,999,950 of them away.

**With parameters → ordering off a menu (`vimAuthenticationContosoAuth`)**
> You hand the waiter a slip: *"Failed sign-ins, last hour, source IP starts with 185.220."* The kitchen reads the slip first, walks past the irrelevant ingredients, and only cooks the three plates you actually want.  
> Sentinel filters on `TimeGenerated` first (last hour → ~80K envelopes survive), then a cheap string check on the IP prefix (~50 envelopes survive), and *only those 50* get fully opened with `parse_json` and turned into ASIM rows.

**Same 50 rows on your screen. The buffet opened 20 million envelopes. The menu opened 50.**

### "But how does the parser *know* what to filter on?" — your skeptic's question, answered

Fair pushback: a function looks like a black box. You'd think calling `vim*(starttime=ago(1h))` would still have to read every envelope and *then* discard the old ones. It doesn't, and here's exactly why — using the two terms we just defined.

**Step 1 — the optimizer inlines the function.** When you write:

```kusto
vimAuthenticationContosoAuth(starttime=ago(1h), srcipaddr_has_any_prefix=dynamic(["185.220."]))
```

…Kusto pastes the parser body into your query. The combined query the optimizer actually plans looks (simplified) like this:

```kusto
ContosoAuthRaw_CL
| where TimeGenerated > ago(1h)                       // ← from the starttime parameter
| extend _srcip = tostring(RawEvent.src.ip)           // ← envelope-opening starts here
| where _srcip startswith "185.220."                  // ← from the srcipaddr parameter
| extend ... full ASIM normalization ...              // ← lots more envelope-opening, on survivors only
```

There is no black box anymore. The optimizer sees a flat pipeline, top to bottom.

**Step 2 — the optimizer pushes the cheap filter all the way to disk.** It reads that pipeline and notices: *"the first `where` only touches `TimeGenerated`, which is an indexed column on disk. I can apply that **before** loading any envelopes at all."* Sentinel jumps straight to the relevant time-shards on disk and only loads ~80K envelopes. The other 19.92M never leave storage.

**Step 3 — the parser author put the next filter in the right place.** The next `where` filters on `_srcip`, which only exists *after* one cheap envelope-peek extracted it. So the optimizer can't push that filter past the loading step — but it *can* apply it before the giant `extend ... full ASIM normalization ...` block. ~80K envelopes get a one-field peek (cheap), ~50 survive, and only those 50 get the full expensive treatment.

**Step 4 — the answer to your question.** The parser doesn't have magical awareness of the data. It has three things:

1. **Cheap columns to filter on early** (`TimeGenerated` directly, plus a quick one-field `_srcip` peek before doing anything else).
2. **An author who put the filters in the right order** — narrowing steps first, expensive steps last.
3. **An optimizer that's allowed to look inside the function** and push those narrowing steps as close to the disk as possible.

If the parser author had written it the wrong way around — full envelope-opening first, `where` afterwards — the optimizer would be stuck. It can't push a filter past a step that creates the very columns the filter needs. The buffet has exactly this problem: by the time you get to filter, all the cooking is already done.

> **The slogan:** *the parser doesn't know your filter; the optimizer does, and the parser body is written in an order that lets the optimizer help.*

### A concrete walk-through with our sample data

We have 20 events including beth's brute-force burst from `185.220.101.42`.

**A hunter exploring at 2 a.m., no idea what they're looking for yet:**
```kusto
ASimAuthenticationContosoAuth | take 100
```
No slip needed. Get everything, look around. *This is what the parameter-less parser is for.*

**The detection rule from Step 5, running every 5 minutes:**
```kusto
vimAuthenticationContosoAuth(
    starttime = ago(1h),
    eventresult = "Failure",
    srcipaddr_has_any_prefix = dynamic(["185.220."]))
| summarize FailCount = count() by TargetUserName, SrcIpAddr
| where FailCount >= 5
```
The slip arrives *before* envelope-opening starts. Sentinel skips ~99.95% of the table on disk. **Same end result as the equivalent buffet query** — just dramatically less I/O and CPU.

You **can** write the same detection logic with the buffet:
```kusto
ASimAuthenticationContosoAuth
| where TimeGenerated > ago(1h)
| where EventResult == "Failure"
| where SrcIpAddr startswith "185.220."
| summarize ...
```
The result is identical. But because `EventResult` and `SrcIpAddr` are columns the buffet *creates* (by opening every envelope), the optimizer can't push those `where` clauses below the parsing step. **The narrowing arrived too late.**

### The performance payoff in one picture

```
 BUFFET (parameter-less)                       MENU (with parameters)
 ───────────────────────                       ───────────────────────
 20,000,000 sealed envelopes on disk           20,000,000 sealed envelopes on disk
         │                                              │
         ▼                                              ▼
   open every envelope (parse_json)              filter on TimeGenerated FIRST
   and build full ASIM columns                   (no envelopes opened yet)
         │                                              │   → 80,000 envelopes loaded
         ▼                                              ▼
   20,000,000 fully-shaped rows                  open each one just enough
   in memory                                     to read _srcip
         │                                              │
         ▼                                              ▼
   apply detection filters                       where _srcip startswith "185.220."
         │                                              │   → 50 envelopes survive
         ▼                                              ▼
   50 rows returned                              fully open & shape only these 50
   Cost: huge                                          │
                                                       ▼
                                                 50 rows returned
                                                 Cost: tiny
```

### Prove it to yourself

In the Logs editor, run both queries against our 20 rows. Click **Query details** in the bottom-right of the results pane. Compare the **"Data scanned"** value.

With 20 rows the difference is negligible — both numbers are tiny. But the *principle* is the same one that decides whether your rule completes in 2 seconds or times out at 10 minutes when the table holds 20 million rows. **The pattern is what matters; the volume just makes it visible.**

---

### Reference: the parameters in detail

You don't need to memorize this. Skim it now, and come back when you're writing your own parser or a detection rule. The parameters fall into three groups.

**1. Time window** (almost always used)

| Parameter | What it does |
|-----------|--------------|
| `starttime` | Drops rows older than this. Filters on indexed `TimeGenerated`. |
| `endtime`   | Drops rows newer than this. Same. |

**2. Identity / network filters** (the "who" and "where")

Our lab parser implements two of these; the full unifier contract has more (see Step 4.8).

| Parameter | What it does | Example |
|-----------|--------------|---------|
| `targetusername_has_any` | Keep only rows where the *target* user (the one being signed into) matches one of these names. | `dynamic(["beth@contoso.nl"])` |
| `srcipaddr_has_any_prefix` | Keep only rows where the **source IP** starts with one of these prefixes. Prefixes, not full IPs, so `["185.220.", "203.0.113."]` covers whole ranges. | `dynamic(["185.220."])` |

> **💡 Why prefixes instead of CIDR:** prefix match is a cheap string operation. CIDR match requires parsing IPs as numbers — slower at scale. ASIM picks the cheap one.

> **💡 Why the 12-param unifier contract has more (`actorusername_has_any`, `srchostname_has_any`, `targetipaddr_has_any_prefix`, `dvcipaddr_has_any_prefix`, `dvchostname_has_any`):** authentication events have multiple roles for "who / where / which device":  
> **src** = where the user is coming **from** (their laptop's public IP)  
> **target** = the resource being signed **into** (e.g., a SharePoint front-end's IP)  
> **dvc** = the system **reporting** the event (the auth gateway log producer)  
> Different rules care about different roles. The unifier's contract covers them all; your `vim*` parser only has to implement the ones that exist in *your* source.

**3. Outcome filters**

| Parameter | What it does | Allowed values |
|-----------|--------------|----------------|
| `eventtype_in` | Only these kinds of authentication events. | `dynamic(["Logon"])`, `dynamic(["Logon","Logoff"])`, `dynamic(["Elevate"])` |
| `eventresult`  | Only this outcome. Single value, not a list. | `"Success"`, `"Failure"`, `"*"` for any |

### The "empty means no filter" convention

| Parameter type | Empty default | What "filled in" means |
|----------------|---------------|------------------------|
| `dynamic` lists (`*_has_any`, `*_in`) | `dynamic([])` → no filter | Keep rows where the column matches any listed value |
| `string` (`eventresult`) | `"*"` → no filter | Keep rows where the column equals exactly this value |
| `datetime` (`starttime`/`endtime`) | `datetime(null)` → no filter | Bound `TimeGenerated` |

That's why every filter in the parser body looks like this:

```kusto
| where (array_length(targetusername_has_any) == 0
         or _upn has_any (targetusername_has_any))
```

Plain English: *"If the slip is blank for this filter, let the row through. Otherwise, only let it through if it matches the slip."*

### Why this is non-negotiable for the unifier

Microsoft's `_Im_Authentication` doesn't know what your users will ask. When a hunter writes:

```kusto
_Im_Authentication(
    starttime=ago(1h),
    targetusername_has_any=dynamic(["beth@contoso.nl"]),
    eventresult="Failure")
```

…the unifier blindly forwards those **named** parameters to every registered parser, including yours. If your parser doesn't accept that exact signature, the call fails — and `union isfuzzy=true` swallows the error (the silent "no rows" trap from the Step 4.8 troubleshooting note).

---

## 4.7 Test the parser

In the **Logs** editor, run:

```kusto
vimAuthenticationContosoAuth(eventresult="Failure")
| summarize FailedAttempts = count() by TargetUserName, SrcIpAddr
| order by FailedAttempts desc
```

You should see the **same brute-force burst** you found in Step 3.7 — `beth@contoso.nl` from `185.220.101.42` with 9 failures. **Same data, same answer, completely different storage strategy.** That's the whole point.

Try a few more calls to feel the parameters:

```kusto
// Last hour only
vimAuthenticationContosoAuth(starttime=ago(1h), endtime=now())

// Specific user, ASIM-shaped output
vimAuthenticationContosoAuth(targetusername_has_any=dynamic(["beth@contoso.nl"]))
| project TimeGenerated, EventResult, SrcIpAddr, SrcGeoCountry

// Logons only, all results
vimAuthenticationContosoAuth(eventtype_in=dynamic(["Logon"]))
```

> **💡 What you can feel right now:** if you change the parser logic and re-save the function, the *same historical data* now returns differently. **You can fix bugs retroactively** — that's the irreversibility flip-side of ingest-time parsing.

---

## 4.8 Register the parser with the unifying parser `_Im_Authentication`

This is the magic step that makes Step 5's detection rule fire on our custom data.

### Two flavors of unifying parser — know which one you have

Sentinel has **two** sets of ASIM unifying parsers, with deliberately different names:

| Flavor | Function name | Available out of the box? |
|--------|---------------|---------------------------|
| **Built-in** (Microsoft-managed, ships in every workspace) | `_Im_Authentication` (leading underscore + capital I) | ✅ Yes |
| **Workspace-deployed** (deploy from [aka.ms/DeployASIM](https://aka.ms/DeployASIM) ARM template) | `imAuthentication` (no underscore) | ❌ Only after deploying |

**For this lab we use the built-in `_Im_Authentication`** — it's already there, no extra deployment.

> **⚠️ If you ever see `Failed to resolve table or column expression named 'imAuthentication'`** — you used the workspace-deployed name without deploying it. Switch to `_Im_Authentication` (or deploy the ARM template).

📖 [Built-in unifying parsers](https://learn.microsoft.com/en-us/azure/sentinel/normalization-about-parsers#unifying-parsers)

### How `Im_AuthenticationCustom` plugs in (no manual wiring needed)

When you query `_Im_Authentication`, Sentinel runs every built-in source-specific parser **plus** — if it exists — a function called **`Im_AuthenticationCustom`**. That convention name is hard-coded into the built-in unifier; we just need to create a function with that exact name.

> **💡 Why a separate "custom" parser instead of editing the built-in one:** the built-in `_Im_Authentication` is owned by Microsoft and gets updated as new sources are added. If you could edit it, your changes would be overwritten by every Sentinel content release. `Im_AuthenticationCustom` is **your** function — the built-in unifier calls it as a contributor. Microsoft can ship updates to its parser without touching your sources, and your custom sources stay registered.

### Create `Im_AuthenticationCustom`

> **⚠️ The signature must match Microsoft's contract exactly.** The built-in `_Im_Authentication` calls `Im_AuthenticationCustom` *positionally* with **12 specific parameters in a specific order**. If your signature doesn't match, the call fails — and because we use `union isfuzzy=true`, the failure is **silent**: you just see no rows, no error. Don't reduce or reorder these parameters even if your inner parser doesn't use them.

1. **Logs** editor → check whether the function already exists by typing `Im_AuthenticationCustom` and hovering. If it doesn't exist, we create it. If it does, we edit it (Functions tab → right-click → **Edit function**).

2. Paste this body (parameters come from the Save dialog, not the body). The body declares 12 inbound params (Microsoft's contract) and forwards only the ones our inner parser actually uses:

   ```kusto
   let DisabledParsers = materialize(
       _GetWatchlist('ASimDisabledParsers')
       | where SearchKey in ('Any', 'ExcludevimAuthenticationContosoAuth')
       | extend SourceSpecificParser = column_ifexists('SourceSpecificParser', '')
       | distinct SourceSpecificParser
   );
   union isfuzzy=true
       vimAuthenticationContosoAuth(
           starttime, endtime,
           targetusername_has_any, srcipaddr_has_any_prefix,
           eventtype_in, eventresult,
           ('ExcludevimAuthenticationContosoAuth' in (DisabledParsers)))
       // add additional vim*Authentication* parsers here as you onboard more sources
   ```

3. **Save as function**:
   - **Function name:** `Im_AuthenticationCustom` (case-sensitive — the built-in unifier looks for this exact name)
   - **Parameters** — add these **12 parameters in this exact order** using **+ Add parameter** for each row (do **not** use the single text-box variant — it silently mangles types):

     | # | Name | Type | Default value |
     |---|------|------|---------------|
     | 1 | `starttime` | `datetime` | `datetime(null)` |
     | 2 | `endtime` | `datetime` | `datetime(null)` |
     | 3 | `targetusername_has_any` | `dynamic` | `dynamic([])` |
     | 4 | `actorusername_has_any` | `dynamic` | `dynamic([])` |
     | 5 | `srcipaddr_has_any_prefix` | `dynamic` | `dynamic([])` |
     | 6 | `srchostname_has_any` | `dynamic` | `dynamic([])` |
     | 7 | `targetipaddr_has_any_prefix` | `dynamic` | `dynamic([])` |
     | 8 | `dvcipaddr_has_any_prefix` | `dynamic` | `dynamic([])` |
     | 9 | `dvchostname_has_any` | `dynamic` | `dynamic([])` |
     | 10 | `eventtype_in` | `dynamic` | `dynamic([])` |
     | 11 | `eventresultdetails_in` | `dynamic` | `dynamic([])` |
     | 12 | `eventresult` | `string` | `"*"` |

> **💡 Why 12 parameters here vs. 7 on `vimAuthenticationContosoAuth`:** the *inner* `vim*` parser is your private contract — it can have any reasonable subset of parameters. But the *unifying* `Im_AuthenticationCustom` has a **fixed contract with Microsoft's built-in unifier**, defined in [the manage-parsers doc](https://learn.microsoft.com/en-us/azure/sentinel/normalization-manage-parsers#add-a-custom-parser-to-a-built-in-unifying-parser). Don't deviate.

> **💡 Debugging tip when "no rows" makes you suspicious:** temporarily flip `union isfuzzy=true` to `union isfuzzy=false` in `Im_AuthenticationCustom` and re-run `_Im_Authentication`. Real errors (wrong arg count, wrong types, missing inner function) will surface instead of being swallowed. Flip it back to `true` once it works.

> **💡 Why `union isfuzzy=true`:** if any one parser in the custom unifier fails (e.g., its underlying table doesn't exist yet because nobody's onboarded that source), `isfuzzy=true` keeps the union working with the parsers that *do* exist. Without it, one missing table breaks every detection.

> **💡 Why the `ASimDisabledParsers` watchlist plumbing:** Microsoft's pattern for emergency disabling individual parsers without editing functions. If a parser is misbehaving in prod (e.g., performance regression), an SOC engineer adds an entry to the `ASimDisabledParsers` watchlist with `SearchKey = ExcludevimAuthenticationContosoAuth`, and the next query run skips it. We're wiring up the same hook so we behave like the production pattern.
> **For this lab the watchlist is empty** — the `_GetWatchlist` call returns no rows, the `in (DisabledParsers)` check is false, and the parser runs normally. You don't need to create the watchlist for the lab to work; the `column_ifexists` guard handles its absence gracefully.

### Verify the unifier picks us up

```kusto
_Im_Authentication
| where EventVendor == "ContosoAuth"
| summarize count() by EventResult, EventType
```

If you see rows — **you're done**. The built-in `_Im_Authentication` is now silently calling `Im_AuthenticationCustom`, which calls our `vimAuthenticationContosoAuth`, which parses the raw blob into ASIM — all in one query, presenting our `ContosoAuth` data alongside whatever Microsoft sources are also wired up.

> **🔍 Explore:** `_Im_Authentication | summarize count() by EventVendor` — see *every* vendor whose parser is plugged into the unifier. In a real environment you'll see Microsoft Entra ID, Windows Security Events, third-party sources, and now `ContosoAuth` sitting beside them.

📖 [`_Im_Authentication` reference](https://learn.microsoft.com/en-us/azure/sentinel/normalization-about-parsers#unifying-parsers)

---

## 4.9 Bonus — complete the pair with `ASimAuthenticationContosoAuth`

> **💡 Skip if you're tight on time.** The detection rule in Step 5 will work without this section. But ASIM's full convention is **two parsers per source per schema**, and finishing the pair takes ~5 minutes.

### Why two parsers, not one?

ASIM defines a **pair** for every source:

| Parser | Has parameters? | Used by |
|--------|-----------------|---------|
| `vimAuthenticationContosoAuth` | Yes (filtering) | Detection rules, scheduled queries, the unifier `_Im_Authentication`. Production paths that benefit from filter-pushdown for performance. |
| `ASimAuthenticationContosoAuth` | No (parameter-less) | Ad-hoc hunting in the Logs editor (`ASimAuthenticationContosoAuth | take 10`). Also called by the parameter-less unifier `_ASim_Authentication`. |

They share **identical body logic**. The parameter-less one is essentially a 1-line wrapper that calls the filtering one with no filters — zero duplication of parsing logic.

> **💡 Analogy refresher:** if `vim*` is the librarian who accepts a request slip, `ASim*` is the same librarian when nobody hands them a slip — they just bring everything. Same person, same translation skills, no filtering applied. Useful when you're exploring and don't yet know what to filter on.

### Create `ASimAuthenticationContosoAuth`

1. **Logs** editor → paste this body:

   ```kusto
   vimAuthenticationContosoAuth()
   ```

   That's the entire body. One line. It calls our filtering parser with no arguments — every parameter takes its default ("no filter"), so we get all rows, fully normalized.

2. **Save** → **Save as function**:
   - **Function name:** `ASimAuthenticationContosoAuth`
   - **Legacy category:** `ASIM`
   - **Parameters:** **none.** Leave the parameters list empty. That's the point.
3. **Save**.

Verify:

```kusto
ASimAuthenticationContosoAuth
| summarize count() by EventResult, EventType
```

Note the lack of `()` — because there are no parameters, you can call it like a table. Hunters love this.

### Register it with the parameter-less custom unifier `ASim_AuthenticationCustom`

The parameter-less built-in unifier is **`_ASim_Authentication`**. Just like `_Im_Authentication` auto-discovers `Im_AuthenticationCustom`, the parameter-less unifier auto-discovers **`ASim_AuthenticationCustom`** (note: no leading underscore on the custom name; **no** `Im_` prefix — it's `ASim_`).

1. **Logs** editor → paste this body:

   ```kusto
   union isfuzzy=true
       ASimAuthenticationContosoAuth
       // add other parameter-less ASim*Authentication* parsers here as you onboard sources
   ```

2. **Save** → **Save as function**:
   - **Function name:** `ASim_AuthenticationCustom` (case-sensitive)
   - **Parameters:** **none.**
3. **Save**.

Verify:

```kusto
_ASim_Authentication
| where EventVendor == "ContosoAuth"
| summarize count() by EventResult, EventType
```

If rows come back — your source is now fully ASIM-compliant on both surfaces.

### Recap of all four functions

```
                    Detection / scheduled rules         Hunters at the keyboard
                                  │                                │
                                  ▼                                ▼
                       _Im_Authentication(…)            _ASim_Authentication
                       (built-in, filtering)            (built-in, parameter-less)
                                  │                                │
                                  ▼                                ▼
                       Im_AuthenticationCustom         ASim_AuthenticationCustom
                       (yours, 12 params)              (yours, no params)
                                  │                                │
                                  ▼                                ▼
                       vimAuthenticationContosoAuth    ASimAuthenticationContosoAuth
                       (yours, filtering)              (yours, parameter-less wrapper)
                                  │                                │
                                  └──────────────┬──────────────┘
                                                 ▼
                                          ContosoAuthRaw_CL
```

Four functions total, but only **two** contain real parsing logic (`vim*` for raw, and we'll add `vim*Ingest` in Step 5 for the ingest-time table). The other two are convention wrappers.

> **🔍 Explore:** if you also want the ingest-time table on the `_ASim_Authentication` surface, repeat the bonus once more in Step 5: create `ASimAuthenticationContosoAuthIngest` (one-liner calling `vimAuthenticationContosoAuthIngest()`) and add it to `ASim_AuthenticationCustom`.

---

## What you built

- [ ] Custom table `ContosoAuthRaw_CL` storing the original nested JSON in a `RawEvent` dynamic column
- [ ] DCR `dcr-contosoauth-raw` with a passthrough transform
- [ ] Bruno request `03 - Send to Raw DCR` returning **204**
- [ ] KQL function `vimAuthenticationContosoAuth` — filtering parser, ASIM Authentication shape
- [ ] Conceptual grasp of what the parameters do (the librarian and the request slip)
- [ ] `Im_AuthenticationCustom` registered with our parser via `union isfuzzy=true`
- [ ] `_Im_Authentication | where EventVendor == "ContosoAuth"` returns rows
- [ ] *(Bonus, optional)* `ASimAuthenticationContosoAuth` wrapper + `ASim_AuthenticationCustom` unifier; `_ASim_Authentication | where EventVendor == "ContosoAuth"` returns rows

✅ **Pipeline 2 of 2 complete.** The same 20 events now exist in two tables, parsed two different ways, and **both** are reachable through `_Im_Authentication`. Step 5 will write one rule that fires on both.

---

[← Back: Step 3](Step3-Ingest-time-Parsing.md) | [Next: Step 5 — ASIM Detection Rule →](Step5-ASIM-Detection-Rule.md)
