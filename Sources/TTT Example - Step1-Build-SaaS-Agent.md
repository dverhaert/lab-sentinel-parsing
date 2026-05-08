# Step 1 — Build a SaaS Agent (Microsoft 365 Copilot)

[← Back to overview](README.md) | [Next: Step 2 — Build a PaaS Agent →](Step2-Build-PaaS-Agent.md)

---

## Table of Contents

- [Goal](#goal)
- [1.1 What is a Declarative Agent?](#11-what-is-a-declarative-agent)
- [1.2 Open Copilot Studio](#12-open-copilot-studio)
- [1.3 Create your Declarative Agent](#13-create-your-declarative-agent)
- [1.4 Add custom instructions](#14-add-custom-instructions)
- [1.5 Add knowledge sources](#15-add-knowledge-sources)
- [1.6 Add an action (optional)](#16-add-an-action-optional)
- [1.7 Test your agent in Copilot](#17-test-your-agent-in-copilot)
- [1.8 Share your agent](#18-share-your-agent)
- [What you built](#what-you-built)

---

## Goal

Create a **declarative agent** in Copilot Studio that runs inside Microsoft 365 Copilot. This is the simplest way to build an agent — no code, no hosting, no infrastructure. The agent uses Copilot's built-in orchestrator and foundation models, which means it inherits all M365 compliance, security, and Responsible AI (RAI) controls by default.

**Difficulty:** Easy  
**Duration:** ~30 minutes  
**Tools:** Browser + Copilot Studio

---

## 1.1 What is a Declarative Agent?

A [declarative agent](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/overview-declarative-agent) extends Microsoft 365 Copilot by adding:

| Component | What it does | Example |
|-----------|-------------|---------|
| **Instructions** | Tell the agent how to behave and respond | "You are a security operations assistant. Always cite your sources." |
| **Knowledge** | Connect data sources the agent can search | SharePoint sites, OneDrive files, uploaded documents |
| **Actions** | Define what the agent can *do* (API calls) | Query an external ticketing system, create a Teams message |

The key difference from a [custom engine agent](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/overview-custom-engine-agent) (Step 2): **you don't bring your own model or hosting**. Copilot handles everything.

> **📖 Learn more:** [Agents for Microsoft 365 Copilot overview](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/agents-overview)

```
┌──────────────────────────────────────┐
│         Microsoft 365 Copilot        │
│  ┌──────────────────────────────┐    │
│  │    Your Declarative Agent     │    │
│  │  ┌────────────────────────┐  │    │
│  │  │  Custom Instructions    │  │    │
│  │  │  Custom Knowledge       │  │    │
│  │  │  Custom Actions         │  │    │
│  │  └────────────────────────┘  │    │
│  │  Uses Copilot's orchestrator  │    │
│  │  Uses Copilot's LLM          │    │
│  └──────────────────────────────┘    │
└──────────────────────────────────────┘
```

---

## 1.2 Open Copilot Studio

1. Navigate to [https://copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Sign in with your Microsoft 365 account
3. Make sure you're in the correct **environment** (top-right corner)

> **Note:** If this is your first time, Copilot Studio may take a moment to set up your environment.
>
> **📖 Learn more:** [Create a declarative agent with Copilot Studio](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/copilot-studio-agent-builder-build)

---

## 1.3 Create your Declarative Agent

1. In Copilot Studio, click **Create** in the left navigation
2. Select **New agent**
3. Choose **Configure manually** (skip the AI-generated setup — we want to understand each piece)
4. Fill in the basics:
   - **Name:** `Security Operations Assistant` (or choose your own)
   - **Description:** `Helps the team find security guidance, triage alerts, and summarize incidents`
   - **Icon:** Pick something recognizable — your team will see this in Teams/Copilot
5. Click **Create**

You now have a blank agent. Let's give it a brain.

---

## 1.4 Add custom instructions

Instructions tell the agent *who it is* and *how to behave*. Think of it as a system prompt.

1. In your agent, go to the **Overview** tab
2. Find the **Instructions** section and click **Edit**
3. Paste the following (or write your own):

```
You are a Security Operations Assistant for our team.

Your role:
- Help team members find relevant security documentation and guidance
- Summarize security incidents and alerts
- Provide step-by-step guidance for common security tasks
- Reference Microsoft security best practices

Your behavior:
- Always cite the source when referencing documentation
- If you're unsure about something, say so — never make up security guidance
- Keep responses concise and actionable
- Use bullet points and tables for clarity
```

4. Click **Save**

> **💡 Tip:** Good instructions are specific about what the agent should and should NOT do. Vague instructions lead to vague responses.

---

## 1.5 Add knowledge sources

Knowledge gives your agent access to data it can search and reference. Without knowledge, the agent only knows what the base Copilot model knows.

1. In your agent, go to the **Knowledge** tab
2. Click **Add knowledge**
3. Choose one or more of these sources:

### Option A: SharePoint site (recommended)
- Select **SharePoint**
- Enter the URL of a SharePoint site your team uses (e.g., your security team's documentation site)
- Copilot will index the content and make it searchable

### Option B: Upload files
- Select **Files**
- Upload relevant documents (PDF, Word, PowerPoint)
- Good candidates: runbooks, incident response procedures, security policies

### Option C: Public website
- Select **Public website**
- Enter a URL — e.g., `https://learn.microsoft.com/en-us/security/`
- The agent can reference content from this site

4. Click **Add** and wait for the knowledge source to be processed

> **🔍 Explore:** Add multiple knowledge sources and then ask the agent questions to see which source it pulls from. Notice how it cites its sources in the response.

---

## 1.6 Add an action (optional)

Actions let your agent *do things* — call APIs, trigger workflows, query external systems. This is optional but demonstrates the full power of declarative agents.

1. In your agent, go to the **Actions** tab
2. Click **Add an action**
3. Choose **Connector** (simplest option)
4. Search for a connector — some good examples:
   - **Microsoft Teams** — Send a message to a channel
   - **Outlook** — Send an email
   - **Planner** — Create a task
5. Select the connector and configure the action
6. Click **Save**

> **Note:** Actions require the connector to be available in your environment. If a connector isn't available, that's fine — skip this step. The agent works perfectly well with just instructions and knowledge.

---

## 1.7 Test your agent in Copilot

> **📖 Learn more:** [Test and publish your declarative agent](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/copilot-studio-agent-builder-publish)

Time to see your agent in action!

1. In Copilot Studio, click the **Test** button (top-right) to open the test pane
2. Try these prompts:
   - `What security incidents should I be aware of?`
   - `How do I respond to a phishing alert?`
   - `Summarize our incident response procedure`
3. Check that the agent:
   - ✅ Follows your custom instructions (tone, format, behavior)
   - ✅ References your knowledge sources (with citations)
   - ✅ Stays within its defined scope

### Test in Microsoft 365 Copilot (Teams)

1. In Copilot Studio, click **Publish** → **Publish** (confirm)
2. After publishing, click **Open in Microsoft 365 Copilot**
3. In Teams or Copilot Chat, find your agent and start chatting
4. Try mentioning the agent with `@Security Operations Assistant` in a Teams chat

---

## 1.8 Share your agent

To let your team use the agent:

1. In Copilot Studio, go to **Settings** → **Security**
2. Under **Sharing**, choose who can use the agent:
   - **Just me** (for testing)
   - **Specific people** (share with teammates)
   - **Everyone in the organization** (requires admin approval)
3. Share the agent link with your team

---

## What you built

| Item | Details |
|------|---------|
| **Agent type** | Declarative agent (SaaS) |
| **Hosting** | Microsoft 365 Copilot infrastructure — no Azure resources needed |
| **Model** | Copilot's built-in foundation model |
| **Identity** | Runs under the user's M365 identity (no separate agent identity yet) |
| **Security** | Inherits all M365 compliance, RAI, and data loss prevention policies |
| **Where it runs** | Teams, Copilot Chat, Outlook, Word, Excel |

### Key takeaway

This agent is **SaaS** — Microsoft hosts everything. You configure *what* the agent does, not *how* it runs. Security and compliance come built-in from M365. But you're limited to Copilot's orchestrator and model, and the agent runs under the **user's identity** (no independent agent identity).

In the next step, you'll build a **PaaS agent** where you control the infrastructure, choose the model, and the agent gets its **own identity** in Entra.

> **Note:** Once your agent is published and approved by your IT admin, it will also appear in the **Agent365 control plane** at [admin.cloud.microsoft](https://admin.cloud.microsoft/#/agents/overview) — the central dashboard for managing all agents in your organization. We'll explore this in [Step 4](Step4-Securing-Agents.md).

---

[← Back to overview](README.md) | [Next: Step 2 — Build a PaaS Agent →](Step2-Build-PaaS-Agent.md)
