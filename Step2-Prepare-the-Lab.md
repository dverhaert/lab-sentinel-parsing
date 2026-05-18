> **Tip:** Always add a unique suffix (like your initials or a random string) to resource names (e.g., `dce-sentinel-parsing-lab-dv`) to avoid conflicts in shared environments.
# Step 2 — Prepare the Lab

[← Back: Step 1](Step1-Why-Parsing-and-ASIM.md) | [Next: Step 3 — Ingest-time Parsing →](Step3-Ingest-time-Parsing.md)

---

## Table of Contents

- [Step 2 — Prepare the Lab](#step-2--prepare-the-lab)
  - [Table of Contents](#table-of-contents)
  - [Goal](#goal)
  - [2.1 What we're building (and why)](#21-what-were-building-and-why)
  - [2.2 Identify your Log Analytics workspace](#22-identify-your-log-analytics-workspace)
  - [2.3 Create a Data Collection Endpoint (DCE)](#23-create-a-data-collection-endpoint-dce)
  - [2.4 Create the Microsoft Entra app registration](#24-create-the-microsoft-entra-app-registration)
    - [Create a client secret](#create-a-client-secret)
  - [2.5 Install Bruno and create a collection](#25-install-bruno-and-create-a-collection)
  - [2.6 Build the OAuth token request in Bruno](#26-build-the-oauth-token-request-in-bruno)
  - [2.7 Get your sample data](#27-get-your-sample-data)
  - [What you built](#what-you-built)

---

## Goal

Set up the **shared infrastructure** that both parsing scenarios (Step 3 and Step 4) will reuse: an app registration, a Data Collection Endpoint, a Bruno collection that can fetch an OAuth token, and the sample JSON.

**Difficulty:** Easy  
**Duration:** ~30 minutes  
**Tools:** Azure portal, Bruno, this repo

---

## 2.1 What we're building (and why)

The **Logs Ingestion API** is the modern, supported way to push custom data into a Log Analytics workspace. It looks like this:

```
   Bruno  ──HTTP POST──►  DCE / DCR endpoint  ──►  DCR  ──►  Custom table
                              ▲
                              │  Bearer token
                              │
                       Microsoft Entra
                       (app registration)
```

Three roles, three reasons:

| Component | Why it exists |
|-----------|---------------|
| **App registration** (with client secret) | Sentinel will not accept anonymous data. You authenticate as the app, and the app must have **`Monitoring Metrics Publisher`** RBAC on each DCR it writes to. This is the security boundary. |
| **Data Collection Endpoint (DCE)** | A regional, network-addressable URL your client POSTs to. Think of it as the "front door" mailbox for ingestion. |
| **Data Collection Rule (DCR)** | The contract: "data arriving on stream `Custom-Foo` should be transformed *like this* and stored in *this table*." We will build two of these in Steps 3 and 4. |

> **💡 Why a DCE separate from a DCR?** The DCR endpoint feature now lets you POST directly to a DCR's URL without a DCE in many cases, but a DCE is still required for some scenarios (private link, certain regions) and is the most universally documented path. **We use a DCE for clarity** — once you understand it, the DCE-less variant is a one-line change.

📖 [Logs Ingestion API overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-ingestion-api-overview)

---

## 2.2 Identify your Log Analytics workspace

Make sure you have a Sentinel-enabled workspace ready, and **note these values** — you'll paste them into Bruno later:

| Value | Where to find it |
|-------|------------------|
| **Subscription ID** | Workspace overview → JSON view |
| **Resource group** | Workspace overview |
| **Workspace name** | Workspace overview |
| **Region** | Workspace overview (e.g., `westeurope`) |
| **Tenant ID** | Microsoft Entra → Overview |

> **💡 Why we collect these now:** every other Azure resource we create (DCE, DCRs, app reg) lives in this region/subscription. Pinning them down upfront prevents region mismatch errors later — DCE and DCR **must be in the same region** as each other (the workspace can be elsewhere, but keep them aligned for simplicity).

---


## 2.3 Create a Data Collection Endpoint (DCE)

1. In the Azure portal, search for **Data Collection Endpoints** → **Create**
2. Fill in:
   - **Subscription / Resource group:** same as your workspace
   - **Endpoint name:** Use a generic and unique name, e.g., `dce-sentinel-parsing-lab-<yourinitials>` (replace `<yourinitials>` with your own to avoid naming collisions in shared environments)
   - **Region:** **same as your workspace**
3. **Review + create** → **Create**
4. Once deployed, open the resource and copy the **Logs ingestion** URL.  
   It looks like: `https://dce-sentinel-parsing-lab-xxxx.westeurope-1.ingest.monitor.azure.com`
5. Paste that URL somewhere temporary — you'll need it as `dceEndpoint` in Bruno.

> 💡 Why "Logs ingestion" URL specifically: a DCE has multiple URLs (logs, metrics, configuration). Logs Ingestion API uses the *Logs ingestion* one. Picking the wrong URL gives a confusing 404.

---

## 2.4 Create the Microsoft Entra app registration

1. Microsoft Entra admin center → **App registrations** → **New registration**
2. **Name:** Use a generic and unique name, e.g., `app-sentinel-parsing-lab-<yourinitials>`
3. **Supported account types:** *Accounts in this organizational directory only* (single tenant)
4. **Redirect URI:** leave blank (we'll never do an interactive login)
5. **Register**
6. On the new app's **Overview** page, copy:
   - **Application (client) ID** → save as `clientId`
   - **Directory (tenant) ID** → save as `tenantId`

### Create a client secret

1. Left nav → **Certificates & secrets** → **+ New client secret**
2. **Description:** `lab-secret`, **Expires:** 90 days (or your preferred duration)
3. **Add**
4. **Immediately copy the `Value` column** (not the Secret ID) → save as `clientSecret`

> 💡 Why now and only now: the secret value is shown exactly once. If you navigate away, you have to delete it and create a new one. Get it into your Bruno env var before you do anything else.

> ⚠️ Security note: treat this secret like a password. Don't paste it into chat, screenshots, or shared docs.

You will assign **`Monitoring Metrics Publisher`** RBAC on the DCRs in Steps 3 and 4 — *after* the DCRs exist. Don't do it now.

📖 [Tutorial: Send data to Logs (portal)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal)

---

## 2.5 Install Bruno and create a collection

[Bruno](https://www.usebruno.com/) is a Postman-like REST client that stores everything as plain text files — perfect for a lab.

1. Install Bruno from <https://www.usebruno.com/downloads>
2. Open Bruno → **Create Collection**
   - **Name:** `Sentinel Parsing Lab`
   - **Location:** anywhere on your disk; just remember it
3. With the collection open, click the **Environments** icon (gear, top-right) → **Configure**
4. Create a new environment named `lab` and add these variables:

| Variable | Value | Mark as secret? |
|----------|-------|-----------------|
| `tenantId` | from Step 2.4 | no |
| `clientId` | from Step 2.4 | no |
| `clientSecret` | from Step 2.4 | **yes** |
| `dceEndpoint` | from Step 2.3 (no trailing slash) | no |
| `dcrImmutableIdIngest` | (leave blank — Step 3 fills it) | no |
| `dcrImmutableIdRaw` | (leave blank — Step 4 fills it) | no |
| `bearerToken` | (leave blank — set automatically by token request) | yes |

5. **Save** the environment, then **select** it (top-right dropdown).

> **💡 Why an environment file:** you never want secrets baked into request files. Bruno's environments live outside the request files, so you can commit your collection to git without leaking secrets.

---

## 2.6 Build the OAuth token request in Bruno

Sentinel's ingestion endpoint expects a **Bearer token**. We get one from Microsoft Entra using the **client credentials flow** — app authenticates with its own secret, no user involved.

1. In Bruno, right-click the collection → **New Request**
   - **Name:** `01 - Get Token`
   - **Type:** `HTTP`
   - **Method:** `POST`
   - **URL:** `https://login.microsoftonline.com/{{tenantId}}/oauth2/v2.0/token`
2. Open the request → **Body** tab → choose **Form URL Encoded** → add these key/value pairs:

| Key | Value |
|-----|-------|
| `client_id` | `{{clientId}}` |
| `client_secret` | `{{clientSecret}}` |
| `grant_type` | `client_credentials` |
| `scope` | `https://monitor.azure.com//.default` |

3. **Headers** tab — add:
   - `Content-Type` : `application/x-www-form-urlencoded`
4. **Tests** (or **Scripts → post-response**) tab — paste this so the token is auto-saved into the env:

   ```js
   const data = res.getBody();
   bru.setEnvVar("bearerToken", data.access_token);
   ```

5. **Save**, then **Send**.
6. You should get a `200 OK` with an `access_token` field in the response. Bruno just stored it as `bearerToken`.

> **💡 Why the double slash in `https://monitor.azure.com//.default`:** that is **not** a typo. Microsoft Entra's v2.0 endpoint expects `<resource>/.default` and the resource for Azure Monitor ingestion happens to end with a `/`, so when concatenated you get `//`. Lots of people fix this "typo" and then spend an hour debugging 401s. Don't be that person.

> **💡 Why client credentials flow:** there is no human user. The app *is* the identity. The combination of `client_id` + `client_secret` proves it's our app; Entra returns a token scoped to whatever permissions the app has been granted on Azure resources.

> **🔍 Explore:** decode the returned `access_token` at <https://jwt.ms> and find your app's `appid`, `tid`, and the audience (`aud`) — `https://monitor.azure.com`. The token's `roles` will be empty until you assign `Monitoring Metrics Publisher` in Step 3.

---

## 2.7 Get your sample data

The repo already includes a sample dataset at:

```
Parsing/sample-data/auth-events.json
```

It contains **20 sign-in events** from a fictional `ContosoAuth` product, with:
- A normal mix of successful and failed sign-ins from real-looking users
- A **brute-force burst** against `beth@contoso.nl` — 9 failed password attempts from a single IP in Bucharest, ending in a "User locked" — which Step 5's detection rule will catch
- A **"signout"** and **"password_change"** event — to prove our parser handles event types other than just sign-ins
- A failed sign-in to a non-existent `administrator@contoso.nl` — classic "no such user" probe

Open the file and skim the structure. Notice the **deeply nested** layout: `event.user.upn`, `event.src.geo.country`, etc. This nesting is exactly what makes parsing necessary — **KQL queries against ASIM expect flat columns**, not nested objects.

> **💡 Why one file for both paths:** the same JSON shape is going through both DCRs. The ingest-time DCR will flatten the nesting via `transformKql`; the query-time DCR will leave it nested and let our KQL function do the flattening on read. Sending identical data to both pipelines is what makes the comparison fair.

---

## What you built

- [ ] Identified your workspace, region, subscription, tenant
- [ ] Created a **DCE** in the workspace's region
- [ ] Created an **app registration** with a **client secret**
- [ ] Installed **Bruno** and created a `Sentinel Parsing Lab` collection
- [ ] Configured an environment with `tenantId`, `clientId`, `clientSecret`, `dceEndpoint`
- [ ] Built and successfully ran a `01 - Get Token` request that auto-stores `bearerToken`
- [ ] Located `Parsing/sample-data/auth-events.json`

✅ **Both Step 3 and Step 4 will reuse all of this. You will not touch the app reg, DCE, or token request again.**

---

[← Back: Step 1](Step1-Why-Parsing-and-ASIM.md) | [Next: Step 3 — Ingest-time Parsing →](Step3-Ingest-time-Parsing.md)
