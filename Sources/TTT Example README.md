# Agent365 — Team Tech Time (TTT)

## Goal of this session

By the end of this TTT session, you will have:

1. **Built a SaaS agent** — A declarative agent in Microsoft 365 Copilot using Copilot Studio (low-code, runs on Microsoft's infrastructure)
2. **Built a PaaS agent** — A prompt agent in Microsoft Foundry Agent Service (portal-based, your own Azure infrastructure)
3. **Explored Agent Identity** — Understood how agents get their own Entra identity (Agent ID) and how to manage it
4. **Applied security controls** — Configured Conditional Access, Purview data security (sensitivity labels, DLP, DSPM for AI), content safety, and reviewed agent activity in audit logs and Defender

The session follows a **build-first, secure-second** approach. You need working agents before you can apply security to them.

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Microsoft 365 / Teams                        │
│                                                                     │
│   ┌───────────────────┐          ┌───────────────────────────────┐  │
│   │  Declarative Agent │          │  Foundry Agent (published)    │  │
│   │  (SaaS - Step 1)   │          │  (PaaS - Step 2)              │  │
│   │                     │          │                               │  │
│   │  Copilot Studio     │          │  Foundry Agent Service        │  │
│   │  Copilot Orchestrator│         │  Custom tools + models       │  │
│   └────────┬────────────┘          └─────────────┬───────────────┘  │
│            │                                      │                  │
└────────────┼──────────────────────────────────────┼──────────────────┘
             │                                      │
             ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Microsoft Entra ID                              │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  Agent ID (Step 3)                                          │   │
│   │  • Agent identity blueprint (type/class of agent)           │   │
│   │  • Agent identity (service principal for the agent)         │   │
│   │  • RBAC role assignments                                    │   │
│   │  • OAuth 2.0 token exchange for tool authentication         │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │  Security controls (Step 4)                                  │   │
│   │  • Conditional Access policies                              │   │
│   │  • Identity Protection                                      │   │
│   │  • Content safety guardrails                                │   │
│   │  • Audit logs + Defender alerts                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Table of Contents

| Step | Title | Difficulty | Duration | Description |
|------|-------|------------|----------|-------------|
| 1 | [Build a SaaS Agent (M365 Copilot)](Step1-Build-SaaS-Agent.md) | Easy | ~30 min | Create a declarative agent in Copilot Studio with custom instructions, knowledge, and actions |
| 2 | [Build a PaaS Agent (Microsoft Foundry)](Step2-Build-PaaS-Agent.md) | Easy–Medium | ~45 min | Create a prompt agent in Foundry Agent Service, add tools, and publish it |
| 3 | [Explore Agent Identity (Entra Agent ID)](Step3-Agent-Identity.md) | Medium | ~30 min | Discover how agents get their own identity, inspect it in Entra, and assign permissions |
| 4 | [Securing Agents with Agent365](Step4-Securing-Agents.md) | Medium | ~60 min | Explore the Agent365 control plane dashboard, manage agents, apply Conditional Access, Purview data security, and content safety |

---

## Prerequisites

Before starting, ensure you have the following:

### Licenses & access
- [ ] Microsoft 365 Copilot license (for Step 1)
- [ ] Azure subscription with permissions to create resources (for Step 2)
- [ ] Access to [Microsoft Entra admin center](https://entra.microsoft.com) (for Steps 3 & 4)
- [ ] Access to [Microsoft Foundry portal](https://ai.azure.com) (for Step 2)
- [ ] Access to [Copilot Studio](https://copilotstudio.microsoft.com) (for Step 1)

### Recommended (not required)
- [ ] Visual Studio Code with [Microsoft 365 Agents Toolkit](https://aka.ms/M365AgentsToolkit) extension
- [ ] Browser with multiple tabs — you'll be switching between portals frequently

### Knowledge
- Basic understanding of Microsoft 365 and Azure
- Familiarity with the Azure Portal
- No coding experience required — all exercises are portal-based

---

## How to use this guide

1. Follow the steps **in order** — each step builds on the previous one
2. Each step has a **"What you built"** section at the end to verify your progress
3. Navigation links at the bottom of each page take you to the next step
4. Exercises marked with **🔍 Explore** are optional deep-dives if you have extra time

---

## Quick links

- [Agent365 control plane (M365 admin center)](https://admin.cloud.microsoft/#/agents/overview)
- [Agents for Microsoft 365 Copilot overview](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/agents-overview)
- [Microsoft Foundry Agent Service overview](https://learn.microsoft.com/en-us/azure/ai-services/agents/overview)
- [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/agent-identity)
- [Microsoft Entra Agent ID documentation](https://learn.microsoft.com/en-us/entra/agent-id/)
- [Agent365 introduction video (Microsoft Mechanics)](https://youtu.be/yWwYLbMvc3s)

---

> **Ready?** Start with [Step 1 — Build a SaaS Agent](Step1-Build-SaaS-Agent.md) →
