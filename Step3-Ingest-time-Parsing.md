> **Tip:** Use a unique suffix for all resource names (e.g., `dcr-auth-ingestdv`, `CustomAuthIngest_CL`) to avoid naming collisions in shared environments.
# Step 3 — Ingest-time Parsing

[← Back: Step 2](Step2-Prepare-the-Lab.md) | [Next: Step 4 — Query-time Parsing →](Step4-Query-time-Parsing.md)

---

## Table of Contents

- [Step 3 — Ingest-time Parsing](#step-3--ingest-time-parsing)
  - [Table of Contents](#table-of-contents)
  - [Goal](#goal)
  - [3.1 What we're building (and why)](#31-what-were-building-and-why)
  - [3.2 Create the custom table + DCR (one click, both at once)](#32-create-the-custom-table--dcr-one-click-both-at-once)
  - [3.3 Add the transform that reshapes the JSON](#33-add-the-transform-that-reshapes-the-json)
  - [3.4 Grant the app registration permission on the DCR](#34-grant-the-app-registration-permission-on-the-dcr)
  - [3.5 Capture the DCR Immutable ID + Stream name](#35-capture-the-dcr-immutable-id--stream-name)
  - [3.6 Build the "send data" request in Bruno](#36-build-the-send-data-request-in-bruno)
  - [3.7 Verify the data in Sentinel](#37-verify-the-data-in-sentinel)
  - [What you built](#what-you-built)

---

## Goal

Build the **first** of our two pipelines: the **ingest-time** path. Data arrives nested, and a DCR transformation flattens and normalizes it **before** it's stored. The resulting table looks like ASIM out of the box.

**Difficulty:** Medium  
**Duration:** ~40 minutes  
**Tools:** Azure portal, Bruno

---

## 3.1 What we're building (and why)

```
Bruno  ──►  DCE  ──►  DCR (with transformKql)  ──►  ContosoAuthIngest_CL
                                │
              source                                       output
              ──────                                       ──────
              {event.user.upn}        ─────►   TargetUserName
              {event.src.ip}          ─────►   SrcIpAddr
              {event.outcome}         ─────►   EventResult
              {event.type}            ─────►   EventType
              ...                                ...
```

The transform is a **mini-KQL query** that runs on every incoming record. It extracts nested values and projects them into ASIM-shaped columns. The original nested blob is **not stored** — only the transform output lands in the table.

> **💡 Why this is a real architectural choice, not just a demo:** for high-volume sources (Defender, AAD, anything with millions of events/day), you do **not** want to pay query-time CPU on every detection run to repeatedly parse the same nested JSON. Doing it once at ingest is dramatically cheaper. The trade-off is that you've baked your schema interpretation into the pipeline — if the source adds a new field next month, your transform won't know about it.

📖 [Structure of a transformation in Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations-kql)

---

## 3.2 Create the custom table + DCR (one click, both at once)

Sentinel's portal can create the table **and** the DCR together. We'll use that.

1. Open your **Log Analytics workspace** in the Azure portal
2. Left nav → **Tables** → **+ Create** → **New custom log (DCR-based)**
3. **Basics:**
    - **Table name:** Use a generic and unique name, e.g., `CustomAuthIngest` (the portal will append `_CL` automatically → final name `CustomAuthIngest_CL`)
    - **Description:** e.g., `Custom authentication sign-in events, normalized at ingest time to ASIM Authentication shape`
    - **Data collection rule:** **Create a new DCR**
       - **Name:** Use a generic and unique name, e.g., `dcr-auth-ingest-<yourinitials>`
       - **Subscription / Resource group:** same as workspace
    - **Data collection endpoint:** select the DCE you created in Step 2 (e.g., `dce-sentinel-parsing-lab-<yourinitials>`)
4. **Next: Schema and transformation**
5. **Upload sample data** — upload `Parsing/sample-data/auth-events.json`. The portal parses it, infers a schema, and shows you a preview.

> **💡 Why the portal needs a sample:** the DCR's `streamDeclarations` block defines what *incoming* data looks like — the columns that Bruno will POST. The portal infers those columns from your sample JSON. We'll then add a `transformKql` that reshapes those incoming columns into the *destination* table's columns.

You'll see the portal infer columns roughly like:
   - `timestamp` (string)
   - `event` (dynamic) ← the whole nested blob

Good. That's exactly what we want as input.

---

## 3.3 Add the transform that reshapes the JSON

In the same wizard step, find the **Transformation editor** and replace whatever is there with this KQL:

```kusto
source
| extend
    TimeGenerated      = todatetime(timestamp),
    EventVendor        = tostring(event.vendor),
    EventProduct       = tostring(event.product),
    EventSchema        = "Authentication",
    EventSchemaVersion = "0.1.4",
    EventCount         = int(1),
    EventStartTime     = todatetime(timestamp),
    EventEndTime       = todatetime(timestamp),
    EventOriginalUid   = tostring(event.id),
    EventOriginalType  = tostring(event.type),
    EventType          = case(
                            tostring(event.type) == "signin",          "Logon",
                            tostring(event.type) == "signout",         "Logoff",
                            tostring(event.type) == "password_change", "Other",
                            "Other"),
    EventResult        = case(
                            tostring(event.outcome) == "Success", "Success",
                            tostring(event.outcome) == "Failure", "Failure",
                            "Other"),
    EventResultDetails = tostring(event.reason),
    TargetUserName     = tostring(event.user.upn),
    TargetUsernameType = "UPN",
    TargetUserId       = tostring(event.user.id),
    SrcIpAddr          = tostring(event.src.ip),
    SrcPortNumber      = toint(event.src.port),
    SrcGeoCountry      = tostring(event.src.geo.country),
    SrcGeoCity         = tostring(event.src.geo.city),
    SrcDvcHostname     = tostring(event.device.hostname),
    SrcDvcOs           = tostring(event.device.os),
    TargetAppName      = tostring(event.app.name),
    TargetSessionId    = tostring(event.app.sessionId)
| project
    TimeGenerated, EventVendor, EventProduct, EventSchema, EventSchemaVersion,
    EventCount, EventStartTime, EventEndTime, EventOriginalUid, EventOriginalType,
    EventType, EventResult, EventResultDetails,
    TargetUserName, TargetUsernameType, TargetUserId,
    SrcIpAddr, SrcPortNumber, SrcGeoCountry, SrcGeoCity,
    SrcDvcHostname, SrcDvcOs,
    TargetAppName, TargetSessionId
```

Let's break down **why** each section exists:

| Section | Why |
|---------|-----|
| `source` | Magic word — this is the input pipeline (the records Bruno POSTs). Always start a transform with `source`. |
| `TimeGenerated = todatetime(timestamp)` | **Mandatory.** Every Log Analytics table must have `TimeGenerated`. Our raw JSON calls it `timestamp`; we rename + cast. |
| The `Event*` block | The five **mandatory ASIM Authentication** fields — `EventVendor`, `EventProduct`, `EventSchema`, `EventSchemaVersion`, plus `EventCount` / `EventStartTime` / `EventEndTime` / `EventType` / `EventResult`. Without these, ASIM detections won't recognize the data. |
| `case(...)` for `EventType` and `EventResult` | ASIM has a **fixed vocabulary**: `EventType` must be `Logon` / `Logoff` / `Elevate`, `EventResult` must be `Success` / `Failure`. Our raw values (`signin`, `signout`, `password_change`, …) don't match — so we map them. **This is normalization in action.** |
| `tostring(event.user.upn)` etc. | Drilling into the nested dynamic blob to extract specific scalar values. `tostring()` casts to a typed column so the destination schema is concrete. |
| `project` at the end | Drops everything that isn't in the ASIM shape. Anything not projected is **discarded forever**. |

> **💡 Why we hard-code `TargetUsernameType = "UPN"`:** ASIM's `UsernameType` logical type expects values like `Windows`, `UPN`, `Email`, `DN`. Our source always emits UPN-style names (`alex@contoso.nl`), so we can safely fix it. If your real source had mixed username styles, you'd `case()` on the value pattern.

> **⚠️ The `case()` for `EventType`:** notice we map `"password_change"` to `"Other"`. ASIM Authentication doesn't have a dedicated `PasswordChange` event type, so `Other` is the documented fallback. Real ASIM purists would route password-change events to a different schema (`AuditEvent`). For this lab, mapping to `Other` keeps everything in one table.

> **➡️ Forward reference for Step 5:** this ingest transform intentionally omits the recommended ASIM fields `SrcDvcIpAddr` and `TargetUserType`; in this lab we backfill those later in the parser layer so the built-in brute-force template can resolve cleanly (see Step 5.1).

7. Click **Apply** in the transformation editor — the preview pane should now show flat ASIM-shaped rows.
8. **Next** through the rest of the wizard, then **Create**.

> **🔍 Explore:** Check the inferred destination schema. The portal generates column types from your `extend` casts — `toint()` makes an `int` column, `todatetime()` makes a `datetime`, etc. Get the casts wrong and you get string columns where you wanted numbers.

---

## 3.4 Grant the app registration permission on the DCR

Until we do this, Bruno will get a `403 Forbidden` when posting.

1. Go to the new DCR resource: `dcr-contosoauth-ingest`
2. Left nav → **Access control (IAM)** → **+ Add** → **Add role assignment**
3. **Role:** `Monitoring Metrics Publisher`
4. **Assign access to:** `User, group, or service principal`
5. **Select members:** search for your app registration name (`app-sentinel-parsing-lab-<yourinitials>`) → select it
6. **Review + assign**

> **💡 Why "Monitoring Metrics Publisher":** despite the name (legacy from the metrics API), this is **the** Logs Ingestion API role. Its single data action `Microsoft.Insights/Telemetry/Write` is what the ingestion endpoint checks. Anything broader (Contributor, Owner) would also work but violates least-privilege.

> **💡 Why on the DCR specifically and not the workspace:** the DCR is the security boundary. The app can write **only** to the DCRs it's been granted on, into **only** the streams those DCRs declare, after **only** the transforms those DCRs define. The workspace itself doesn't need to know the app exists.

> **⏳ Note:** RBAC propagation takes 1-3 minutes. If your first POST attempt 403s, wait and retry.

---

## 3.5 Capture the DCR Immutable ID + Stream name

We need two values from the DCR for the Bruno request URL.

1. On the DCR resource page, click **JSON View** (top-right)
2. In the JSON, find:
   - `properties.immutableId` → looks like `dcr-abc123def456...` → copy as `dcrImmutableIdIngest` in your Bruno env
   - `properties.dataFlows[0].streams[0]` → looks like `Custom-ContosoAuthIngest_CL` → that's your **stream name**

Update your Bruno `lab` environment:

| Variable | Value |
|----------|-------|
| `dcrImmutableIdIngest` | `dcr-abc123...` (whatever you copied) |
| `streamIngest` | `Custom-ContosoAuthIngest_CL` |

> **💡 Why "immutable" ID:** the friendly DCR name can be renamed; the immutable ID is permanent and is what the ingestion endpoint matches against the JWT's permissions. Rename-safe by design.

---

## 3.6 Build the "send data" request in Bruno

1. New request in the collection:
   - **Name:** `02 - Send to Ingest DCR`
   - **Method:** `POST`
   - **URL:**

     ```
     {{dceEndpoint}}/dataCollectionRules/{{dcrImmutableIdIngest}}/streams/{{streamIngest}}?api-version=2023-01-01
     ```

2. **Headers:**

   | Key | Value |
   |-----|-------|
   | `Authorization` | `Bearer {{bearerToken}}` |
   | `Content-Type` | `application/json` |

3. **Body** → **JSON** → paste the **entire content** of `Parsing/sample-data/auth-events.json`.  
   The body is a JSON array. The Logs Ingestion API expects an array of records — exactly the shape our file already has.

4. **Save** → **Send**.
5. Expected response: **`204 No Content`**. (Yes, *no body* is the success signal.)

> **💡 Why 204 and not 200:** the API ack's intake but never echoes the data back. If you get `400`, your JSON doesn't match the stream declaration. If `401`, your token expired (re-run `01 - Get Token`). If `403`, RBAC hasn't propagated yet (wait a minute). If `404`, double-check the DCR Immutable ID and stream name in the URL.

---

## 3.7 Verify the data in Sentinel

Data takes **2-5 minutes** to appear after a successful 204. Be patient.

1. Open the **Log Analytics workspace** → **Logs**
2. Run:

   ```kusto
   ContosoAuthIngest_CL
   | take 50
   ```

3. You should see ~20 rows with **flat ASIM-shaped columns** — `TargetUserName`, `SrcIpAddr`, `EventResult`, `EventType`, etc.
4. Confirm the brute-force burst is there:

   ```kusto
   ContosoAuthIngest_CL
   | where EventResult == "Failure"
   | summarize FailedAttempts = count() by TargetUserName, SrcIpAddr
   | order by FailedAttempts desc
   ```

   `beth@contoso.nl` from `185.220.101.42` should be at the top with **9 failures**.

> **💡 What just happened philosophically:** there is no nested `event` column in this table. The original raw JSON shape **does not exist** anywhere in Sentinel anymore. If we had transformed something incorrectly, we couldn't go back and fix it for these 20 records — we'd have to re-send them. That's the **irreversibility** trade-off of ingest-time parsing in concrete form.

> **🔍 Explore:** open the `ContosoAuthIngest_CL` table schema in the workspace's **Tables** blade. Note that the columns match what you `project`-ed in the transform. Anything not projected (the raw `event` blob, `timestamp` as a string, etc.) was dropped.

---

## What you built

- [ ] Custom table `ContosoAuthIngest_CL` (DCR-based)
- [ ] DCR `dcr-contosoauth-ingest` with a `transformKql` that maps nested JSON to ASIM Authentication columns
- [ ] `Monitoring Metrics Publisher` granted to the app on the DCR
- [ ] Bruno env updated with `dcrImmutableIdIngest` and `streamIngest`
- [ ] Bruno request `02 - Send to Ingest DCR` returning **204 No Content**
- [ ] 20 ASIM-shaped rows visible in `ContosoAuthIngest_CL`
- [ ] Brute-force burst confirmed by KQL summarize

✅ **Pipeline 1 of 2 complete.** Ingest-time parsing is done. Take a stretch break — Step 4 builds Pipeline 2 with the same data, but parses it on read.

---

[← Back: Step 2](Step2-Prepare-the-Lab.md) | [Next: Step 4 — Query-time Parsing →](Step4-Query-time-Parsing.md)
