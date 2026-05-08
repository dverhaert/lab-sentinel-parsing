# Step 3 — Explore Agent Identity (Microsoft Entra Agent ID)

[← Back to Step 2](Step2-Build-PaaS-Agent.md) | [Next: Step 4 — Securing Agents →](Step4-Securing-Agents.md)

---

## Table of Contents

- [Goal](#goal)
- [3.1 Why do agents need their own identity?](#31-why-do-agents-need-their-own-identity)
- [3.2 Find your agent's identity in the Azure Portal](#32-find-your-agents-identity-in-the-azure-portal)
- [3.3 Explore Agent ID in the Entra admin center](#33-explore-agent-id-in-the-entra-admin-center)
- [3.4 Understand the identity components](#34-understand-the-identity-components)
- [3.5 Assign RBAC permissions to your agent](#35-assign-rbac-permissions-to-your-agent)
- [3.6 Understand how tool authentication works](#36-understand-how-tool-authentication-works)
- [3.7 Explore: Access packages and lifecycle workflows](#37-explore-access-packages-and-lifecycle-workflows)
- [What you built](#what-you-built)

---

## Goal

Discover how [Microsoft Entra Agent ID](https://learn.microsoft.com/en-us/entra/agent-id/) gives AI agents their own identity — separate from human users and traditional service principals. You'll inspect the identity that was automatically created when you published your Foundry agent in Step 2, and assign it permissions to access Azure resources.

**Difficulty:** Medium  
**Duration:** ~30 minutes  
**Tools:** Browser + Azure Portal + Entra admin center

> **Note:** Agent ID is currently in preview. Everything in this step works, but the UI and capabilities may evolve.

---

## 3.1 Why do agents need their own identity?

In Step 1, your declarative agent ran under the **user's identity** — it could only access what the user could access. That seems safe, but it limits the agent.

In Step 2, your Foundry agent got its **own identity**. Why does that matter?

| Problem | Without Agent Identity | With Agent Identity |
|---|---|---|
| Who did this? | "A user did something" — but was it the user or their agent? | Clear distinction: agent actions are logged separately |
| Access control | Agent inherits all user permissions (too broad) | Agent gets scoped, least-privilege permissions |
| Governance | Can't apply policies to agents specifically | Conditional Access, Identity Protection, etc. target agents |
| Scale | One user = one set of permissions | Hundreds of agents, each with right-sized access |
| Revocation | Disable the user = disable the agent (too drastic) | Disable the agent identity independently |

### The three identity types in Entra

```
Microsoft Entra ID
├── Workforce identities     → Humans (employees, guests)
├── Workload identities      → Apps, services, managed identities
└── Agent identities (NEW)   → AI agents ← this is what we're exploring
```

Agent identities are a **new identity class** — distinct from both workforce and workload identities. This lets Entra apply agent-specific security controls.

> **📖 Learn more:** [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)

---

## 3.2 Find your agent's identity in the Azure Portal

When you published your agent in Step 2, Foundry automatically created:
- An **agent identity blueprint** (the "type" of agent)
- An **agent identity** (the specific service principal)

Let's find them.

1. Navigate to [https://portal.azure.com](https://portal.azure.com)
2. Go to your **resource group** where the Foundry project lives
3. Find and click on your **Foundry project** resource (type: `Microsoft.MachineLearningServices/workspaces`)
4. On the **Overview** pane, click **JSON View** (top-right)
5. Select the **latest API version** from the dropdown
6. In the JSON, look for the identity section. You'll find:

```json
{
  "properties": {
    "agentIdentityBlueprintId": "/subscriptions/.../agentIdentityBlueprints/...",
    "agentIdentityId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```

7. **Copy the `agentIdentityId`** — you'll need this in Step 3.5 to assign RBAC roles

> **💡 Shared vs. Distinct identity:**  
> - The **project-level** JSON view shows the *shared* project identity (used by all unpublished agents in this project)  
> - Your **published agent's** application resource has its own *distinct* identity (find it by navigating to the agent application resource → JSON View)

---

## 3.3 Explore Agent ID in the Entra admin center

The Entra admin center has a dedicated section for managing all agent identities across your tenant.

1. Navigate to [https://entra.microsoft.com](https://entra.microsoft.com)
2. In the left nav, go to **Identity** → **Agent ID** → **All agent identities**
3. You should see an inventory of all agent identities in your tenant, including:
   - Your Foundry agent from Step 2
   - Any Copilot Studio agents
   - Any other agents registered in the tenant

4. Click on your agent identity to explore:
   - **Overview:** Name, type, application ID, creation date
   - **Permissions:** What the agent has access to
   - **Activity:** Sign-in and audit logs for this agent
   - **Conditional Access:** (We'll configure this in Step 4)

> **🔍 Explore:** Look at other agent identities in the list. Are there any you didn't expect? Copilot Studio agents, third-party agents, or system-created agents may already be present.

---

## 3.4 Understand the identity components

Here's how the pieces fit together:

```
Agent Identity Blueprint (the "class")
│
│   Defines: agent type, publisher, OAuth credentials
│   Example: "Security Incident Summarizer" by your org
│
└── Agent Identity (the "instance")
    │
    │   Is: a service principal in Entra
    │   Has: its own application ID, object ID
    │   Can: be assigned RBAC roles, targeted by CA policies
    │
    └── Federated Credential
        │
        │   Links to: Foundry project's managed identity
        │   Purpose: no stored secrets — Azure handles rotation
        │
        └── Runtime Token Exchange
            │
            ├── 1. Blueprint authenticates to Entra (via managed identity)
            ├── 2. Entra issues agent identity token
            ├── 3. Agent requests scoped token for target resource
            └── 4. Tool call is made with scoped token
```

The key insight: **no secrets are stored in the agent's configuration**. Authentication flows through federated credentials and Azure's managed identity infrastructure.

> **📖 Learn more:** [MCP server authentication with agent identity](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/mcp-authentication) | [Agent-to-Agent authentication](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-to-agent-authentication)

### Attended vs. Unattended access

| Flow | How it works | When to use |
|------|-------------|-------------|
| **Attended (OBO)** | Agent acts on behalf of a signed-in user. Uses OAuth 2.0 On-Behalf-Of flow. Agent can only access what the user has consented to. | User-facing agents in Teams — "do this for me" |
| **Unattended (App-only)** | Agent acts under its own authority. Uses OAuth 2.0 client credentials flow. Access governed by agent's own RBAC roles. | Background/automated agents — "do this on your own" |

---

## 3.5 Assign RBAC permissions to your agent

Let's give your agent permission to access an Azure resource. We'll use a Storage Account as an example — but this works for any Azure resource.

### Exercise: Grant your agent read access to a Storage Account

1. **Create a Storage Account** (or use an existing one):
   - In the Azure Portal, go to **Storage accounts** → **Create**
   - Name: `agent365tttstore` (or your own — must be globally unique)
   - Keep defaults, click **Create**

2. **Assign a role to the agent identity:**
   - In your Storage Account, go to **Access control (IAM)**
   - Click **Add** → **Add role assignment**
   - **Role:** `Storage Blob Data Reader` (keep it least-privilege!)
   - **Members:** Select **User, group, or service principal**
   - Search for your agent's `agentIdentityId` (the one you copied in 3.2) or search by agent name
   - Click **Select** → **Review + assign**

3. **Verify the assignment:**
   - Still in **Access control (IAM)**, go to **Role assignments** tab
   - Find your agent identity in the list
   - Confirm it has `Storage Blob Data Reader`

> **Important:** If you publish or re-publish the agent later, a **new** distinct agent identity is created. You'd need to repeat this role assignment for the new identity. The old assignments don't carry over.

> **🔍 Explore:** What happens if you assign a role to the *shared project identity* instead? All unpublished agents in the project would inherit that access. Think about the blast radius implications.

---

## 3.6 Understand how tool authentication works

When your agent calls a tool (like an MCP server that reads from Azure Storage), this is what happens behind the scenes:

```
Agent invokes "read blob" tool
         │
         ▼
┌─────────────────────────┐
│ 1. Blueprint Auth        │  Foundry presents the blueprint's
│                          │  federated credential to Entra
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ 2. Agent Identity Token  │  Entra validates and issues a
│                          │  token for the agent identity
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ 3. Scoped Token Request  │  Foundry requests a new token
│                          │  scoped to audience:
│                          │  https://storage.azure.com
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│ 4. Authenticated Call    │  Tool makes the API call with
│                          │  the scoped access token.
│                          │  Storage validates token + RBAC.
└─────────────────────────┘
```

**Common audiences for Azure services:**

| Service | Audience value |
|---------|---------------|
| Azure Storage | `https://storage.azure.com` |
| Microsoft Graph | `https://graph.microsoft.com` |
| Azure Key Vault | `https://vault.azure.net` |
| Azure Logic Apps | `https://logic.azure.com` |
| Azure Cosmos DB | `https://cosmos.azure.com` |

> **Gotcha:** An incorrect audience value causes authentication failures even when RBAC roles are correctly assigned. The audience must match the resource identifier of the downstream service, not the URL of the tool itself.

> **📖 Learn more:** [Azure role-based access control in Foundry](https://learn.microsoft.com/en-us/azure/foundry/concepts/rbac-foundry)

---

## 3.7 Explore: Access packages and lifecycle workflows

> **🔍 This section is optional.** It covers governance capabilities for managing agent identities at scale. Skip ahead to [What you built](#what-you-built) if you're short on time.

### Access packages for agents (Entitlement Management)

Instead of manually assigning RBAC to each agent, you can create **access packages** — pre-bundled sets of permissions that agents can request, with approval workflows.

1. Navigate to [https://entra.microsoft.com](https://entra.microsoft.com)
2. Go to **Identity Governance** → **Entitlement Management** → **Access packages**
3. Click **New access package**
4. Configure:
   - **Name:** `Agent - Storage Reader Access`
   - **Description:** `Grants agents read access to blob storage`
   - **Resource roles:** Add the Storage Blob Data Reader role
5. Under **Policy**, set:
   - **Requestors:** Agent identities (or specific agents)
   - **Approval:** Require sponsor approval (the agent's designated owner reviews and approves)
   - **Expiration:** Set a time limit (e.g., 90 days) — access doesn't last forever
6. Click **Create**

This is the same Entitlement Management you likely already use for human users — now applied to agent identities. A sponsor (typically the agent's owner or a security admin) reviews and approves access requests from agents.

### Lifecycle workflows for ownerless agents

What happens when an agent's owner leaves the organization? The agent keeps running with all its permissions — a security risk.

**Lifecycle workflows** can detect and self-remediate this:

1. In Entra admin center, go to **Identity Governance** → **Lifecycle Workflows**
2. Explore or create a workflow with:
   - **Trigger:** Agent owner is disabled/deleted (ownership change event)
   - **Action:** Notify the security team that an agent is now ownerless
   - **Escalation:** If no new owner is assigned within X days, disable the agent identity

This prevents "orphaned agents" from accumulating in your tenant with stale permissions.

### Entra Agent ID dashboard metrics

Back in the Entra admin center under **Identity** → **Agent ID** → **Overview**, look for:

- **Total agent identities** in your tenant
- **Agents by platform** (Copilot Studio, Foundry, third-party)
- **Risky agents** flagged by Identity Protection
- **Ownerless agents** that need attention
- **Recent activity** trends

> **💡 Tip:** A healthy agent estate has clear ownership, time-bounded permissions, and no orphaned identities. These governance tools help you maintain that as your agent count grows from a handful to hundreds.

---

## What you built

| Item | Details |
|------|---------|
| **Explored** | Agent identity blueprint + agent identity in Azure Portal |
| **Discovered** | All agent identities in Entra admin center → Agent ID |
| **Understood** | The 4-step OAuth token exchange for tool authentication |
| **Configured** | RBAC role assignment (Storage Blob Data Reader) to agent identity |

### Key takeaway

Agent identities are a **new first-class identity type** in Microsoft Entra. They solve the fundamental problem of "who is doing what" when AI agents access enterprise resources. By giving each agent its own identity:

- You can **audit** agent activity separately from user activity
- You can **scope** agent permissions with least-privilege RBAC
- You can **govern** agents at scale through blueprints
- You can **apply security policies** specifically to agents

That last point is what Step 4 is all about.

---

[← Back to Step 2](Step2-Build-PaaS-Agent.md) | [Next: Step 4 — Securing Agents →](Step4-Securing-Agents.md)
