# Step 2 — Build a PaaS Agent (Microsoft Foundry Agent Service)

[← Back to Step 1](Step1-Build-SaaS-Agent.md) | [Next: Step 3 — Agent Identity →](Step3-Agent-Identity.md)

---

## Table of Contents

- [Goal](#goal)
- [2.1 SaaS vs PaaS — What's the difference?](#21-saas-vs-paas--whats-the-difference)
- [2.2 Set up a Foundry project](#22-set-up-a-foundry-project)
- [2.3 Create a Prompt Agent](#23-create-a-prompt-agent)
- [2.4 Add tools to your agent](#24-add-tools-to-your-agent)
- [2.5 Test your agent in the playground](#25-test-your-agent-in-the-playground)
- [2.6 Publish your agent](#26-publish-your-agent)
- [2.7 Share to Teams / M365 Copilot (optional)](#27-share-to-teams--m365-copilot-optional)
- [What you built](#what-you-built)

---

## Goal

Create a **prompt agent** in [Microsoft Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/overview). Unlike the declarative agent from Step 1, this agent runs on **your Azure infrastructure** — you choose the model, add tools, and when you publish it, the agent gets its **own identity** in Microsoft Entra (which we'll explore in Step 3).

**Difficulty:** Easy–Medium  
**Duration:** ~45 minutes  
**Tools:** Browser + Azure Portal + Foundry Portal

---

## 2.1 SaaS vs PaaS — What's the difference?

You built a SaaS agent in Step 1. Now you're building a PaaS agent. Here's the comparison:

| | Step 1: Declarative Agent (SaaS) | Step 2: Foundry Agent (PaaS) |
|---|---|---|
| **Hosting** | Microsoft 365 infrastructure | Your Azure subscription |
| **Model** | Copilot's built-in model | Your choice (GPT-4o, Llama, DeepSeek, etc.) |
| **Orchestration** | Copilot's orchestrator | Foundry Agent Service runtime |
| **Identity** | User's M365 identity | Dedicated agent identity in Entra |
| **Tools** | Connectors + plugins | Web search, file search, code interpreter, MCP servers, custom functions |
| **Cost** | Included in Copilot license | Azure consumption (pay-per-use) |
| **Control** | Limited — configure, not code | Full — portal or SDK |
| **Where it runs** | M365 apps only | M365 apps + external apps + API endpoints |

**When to choose PaaS:** You need a specific model, custom tools, agent-to-agent communication, or an independent agent identity with its own permissions.

---

## 2.2 Set up a Foundry project

A Foundry project is the workspace where your agents live. It provides shared resources (models, tools, identity) for all agents within it.

> **📖 Learn more:** [Set up your Foundry environment](https://learn.microsoft.com/en-us/azure/foundry/agents/environment-setup)

### Create a Foundry hub + project

1. Navigate to [https://ai.azure.com](https://ai.azure.com) and sign in
2. Click **Create project**
3. Fill in:
   - **Project name:** `Agent365-TTT` (or your own)
   - **Hub:** Create a new hub or select an existing one
     - If creating new: choose your subscription, resource group, and region
   - **Region:** Pick a region that supports Agent Service (West US, East US, Sweden Central, etc.)
4. Click **Create**
5. Wait for the project to provision (this may take a few minutes)

> **Note:** When the project is created, Foundry automatically provisions a default **agent identity blueprint** and **agent identity** in Entra. You won't see this yet — we'll explore it in Step 3.

### Deploy a model

Your agent needs a language model to reason with.

1. In your Foundry project, go to **Model catalog** in the left nav
2. Search for **GPT-4o** (or another model you prefer)
3. Click **Deploy** → **Deploy to project**
4. Keep the default deployment settings
5. Click **Deploy**
6. Once deployed, note the **deployment name** — you'll select this when creating your agent

---

## 2.3 Create a Prompt Agent

A [prompt agent](https://learn.microsoft.com/en-us/azure/foundry/agents/overview#prompt-agents) is defined entirely through configuration — instructions, model selection, and tools. No code required.

1. In your Foundry project, go to **Agents** in the left nav
2. Click **New agent**
3. Configure:
   - **Name:** `Security Incident Summarizer`
   - **Model:** Select the model you deployed in 2.2
   - **Instructions:** Paste the following:

```
You are a Security Incident Summarizer agent.

Your role:
- Analyze security incident data provided by the user
- Produce concise, structured incident summaries
- Identify key indicators of compromise (IOCs)
- Suggest initial triage steps based on the incident type
- Search the web for the latest threat intelligence when relevant

Output format:
- Always start with a one-line severity assessment (Critical / High / Medium / Low)
- Follow with a structured summary: What happened, What's affected, Recommended actions
- Use tables for IOCs and affected assets
- Keep responses under 500 words unless asked for more detail

Constraints:
- Never fabricate IOCs or CVE numbers — if you're unsure, say so
- Always indicate when information comes from web search vs. provided data
```

4. Click **Create**

---

## 2.4 Add tools to your agent

Tools give your agent capabilities beyond text generation. Foundry provides several built-in tools.

> **📖 Learn more:** [Foundry tool catalog](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-catalog)

1. In your agent's configuration, find the **Tools** section
2. Click **Add tool** and enable the following:

### Web Search (Bing Grounding)
- Toggle **On**
- This allows your agent to search the web for current threat intelligence, CVE details, and Microsoft security documentation

### Code Interpreter
- Toggle **On**
- This lets the agent write and execute Python code — useful for parsing log files, calculating statistics, or generating charts from incident data

### File Search (optional)
- Toggle **On** if you want to upload files (e.g., sample log files, incident templates) that the agent can search through
- To add files: click **Add data source** → **Upload files** → select your files

### MCP Server (optional, for extra credit)
- Click **Add tool** → **MCP Server**
- You can connect external MCP servers that provide additional capabilities
- Example: Azure DevOps MCP Server to query work items related to security incidents

3. Click **Save**

> **💡 Tip:** Start with just Web Search and Code Interpreter. You can always add more tools later.

---

## 2.5 Test your agent in the playground

1. In your agent's page, click **Try in playground** (or navigate to the **Agents playground**)
2. Your agent should be pre-selected — verify the model and tools are listed
3. Try these prompts:

**Basic test:**
```
I received an alert about a suspicious login from IP 203.0.113.42 to our 
admin portal at 3:00 AM. The user account is admin@contoso.com. 
What should I do?
```

**Web search test:**
```
Search for the latest information about CVE-2024-3400 and summarize the 
threat, affected products, and recommended mitigations.
```

**Code interpreter test:**
```
Here are failed login counts by hour for the last 24 hours:
[0,0,0,2,45,120,89,12,3,1,0,0,5,8,7,4,3,2,50,95,110,45,10,2]

Analyze this data. Are there any anomalous spikes? Create a chart showing 
the pattern and highlight the suspicious hours.
```

4. Verify:
   - ✅ The agent follows the instruction format (severity → summary → actions)
   - ✅ Web search results are cited
   - ✅ Code interpreter produces a chart/analysis
   - ✅ The agent doesn't fabricate CVE numbers

### Inspect agent tracing

1. After a conversation, click **View trace** (or go to **Tracing** in the left nav)
2. You can see every step the agent took:
   - Model calls (what prompt was sent, what response came back)
   - Tool invocations (which tools were called and with what parameters)
   - Decision points (why the agent chose to use a specific tool)

> **🔍 Explore:** Compare the agent's behavior with and without tools enabled. Disable Web Search and ask about a recent CVE — notice how the agent either says it doesn't know or gives outdated information.

---

## 2.6 Publish your agent

Publishing creates a **stable endpoint** and — critically for Step 3 — a **distinct [agent identity](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)** separate from the shared project identity.

> **📖 Learn more:** [Publish and share agents in Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-agent)

1. In your agent's page, click **Publish**
2. Review the publishing configuration:
   - **Version:** Automatically assigned
   - **Endpoint:** A stable URL will be generated for programmatic access
3. Click **Publish**
4. After publishing, note:
   - The agent now has its own **API endpoint**
   - A new **agent identity blueprint** and **agent identity** have been created in Entra (we'll look at these in Step 3)

> **Important:** When you publish, the agent switches from the **shared project identity** to a **distinct agent identity**. Any RBAC roles assigned to the shared project identity do NOT carry over — you'd need to reassign them to the new identity.

---

## 2.7 Share to Teams / M365 Copilot (optional)

You can make your Foundry agent available in the same places as your Step 1 agent.

1. In your agent's page, go to **Share** or **Channels**
2. Select **Microsoft Teams** or **Microsoft 365 Copilot**
3. Follow the prompts to register your agent
4. Once published, users can find and interact with your agent in Teams

This is where SaaS and PaaS agents converge — **both can appear in Teams**, but under the hood they're fundamentally different in hosting, identity, and control.

> **📖 Learn more:** [Publish agents to Microsoft 365 Copilot and Teams](https://learn.microsoft.com/en-us/azure/foundry/agents/how-to/publish-copilot)

> **Note:** Once published, your Foundry agent will also show up in the **Agent365 control plane** at [admin.cloud.microsoft](https://admin.cloud.microsoft/#/agents/overview), alongside agents built in Copilot Studio and other platforms. We'll explore this in [Step 4](Step4-Securing-Agents.md).

---

## What you built

| Item | Details |
|------|---------|
| **Agent type** | Prompt agent (PaaS) |
| **Hosting** | Microsoft Foundry Agent Service — your Azure subscription |
| **Model** | Your choice (e.g., GPT-4o) from the Foundry model catalog |
| **Identity** | Dedicated agent identity in Microsoft Entra (created on publish) |
| **Tools** | Web Search, Code Interpreter, optionally File Search and MCP |
| **Security** | Your responsibility — content filters built-in, but RBAC/network/CA are yours to configure |
| **Where it runs** | Foundry playground, API endpoint, optionally Teams/M365 Copilot |

### Key takeaway

This agent is **PaaS** — you own the Azure infrastructure, choose the model, and the agent gets its own Entra identity. You have more control, but also more responsibility for security. That's exactly what we'll address in Steps 3 and 4.

### SaaS vs PaaS — The security gap

| Security aspect | SaaS (Step 1) | PaaS (Step 2) |
|---|---|---|
| Compliance & RAI | ✅ Inherited from M365 | ⚠️ You configure it |
| Agent identity | ❌ Uses user's identity | ✅ Has its own identity |
| Network isolation | ✅ Microsoft managed | ⚠️ You configure VNet |
| Content safety | ✅ Built-in | ✅ Built-in (configurable) |
| Conditional Access | ❌ N/A (user's policies apply) | ✅ Can target agent identity |
| Audit trail | Limited to M365 audit | ✅ Full agent-level audit |

The PaaS agent has more security *capabilities* but also more security *responsibility*. In the next step, we'll explore the agent's identity.

---

[← Back to Step 1](Step1-Build-SaaS-Agent.md) | [Next: Step 3 — Agent Identity →](Step3-Agent-Identity.md)
