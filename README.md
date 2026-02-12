# 🎨 Artyfacts

**Agent artifacts that you can actually read.**

Artyfacts is a storage and viewing layer for AI agent outputs. Stop polluting your codebase with random markdown files. Stop losing artifacts on local machines. Start seeing what your agents actually produce.

---

## The Problem

AI agents produce a lot of stuff: research documents, analysis, specs, code, plans. Today, these artifacts end up:

- 📁 **Dumped in codebases** — mixed alongside real code, cluttering your repo
- 💻 **Trapped on local machines** — inaccessible when you're not at that computer  
- 🔗 **Unconnected** — no way to trace which artifact led to which decision
- 🗑️ **Unmanaged** — no lifecycle, no cleanup, everything permanent by default

![Agent artifacts polluting a codebase](./assets/artifact-pollution.png)

## The Solution

Artyfacts gives your agents a proper home for their outputs:

```typescript
import { Artyfacts } from '@artyfacts/sdk';

const artyfacts = new Artyfacts({ apiKey: process.env.ARTYFACTS_API_KEY });

// Agent uploads artifact
const artifact = await artyfacts.upload({
  type: 'document/markdown',
  title: 'Competitor Analysis',
  content: analysisMarkdown,
  source: {
    agentId: 'research-agent',
    taskId: 'ART-456'
  }
});

// Returns a shareable URL
console.log(artifact.url);
// → https://artyfacts.dev/a/7f3b2a1c
```

Human clicks the link → sees beautifully rendered artifact. No GitHub. No local files. No mess.

---

## Features

### 📄 Beautiful Rendering
Markdown, code, JSON, images — all rendered beautifully in the browser. Not raw files. Not GitHub's code review UI. Actually readable.

### 🔗 Shareable Links
Every artifact gets a URL. Share it in Slack, Linear, email. Links unfurl with previews.

### 🌳 Lineage Tracking
See how artifacts connect. Research → Analysis → Tickets → Code. Trace the full path from idea to implementation.

### ⏱️ Lifecycle Management
Not everything needs to live forever. Set retention policies: ephemeral (24h), short-term (7d), long-term (90d), or permanent.

### 🔄 Versioning
Track changes to artifacts over time. See diffs. Pin specific versions for handoffs.

### 🤝 Agent-to-Agent Handoffs
Built-in protocol for passing artifacts between agents with context, expectations, and deadlines.

---

## How It Works

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Agent     │────▶│  Artyfacts  │────▶│   Human     │
│  produces   │     │   stores    │     │   reviews   │
│  artifact   │     │  & renders  │     │  & approves │
└─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Another   │
                    │   Agent     │
                    │  continues  │
                    └─────────────┘
```

1. **Agent finishes work** → uploads artifact to Artyfacts
2. **Artyfacts stores it** → returns shareable URL
3. **Agent reports completion** → includes URL in output
4. **Human clicks link** → sees beautifully rendered artifact
5. **Another agent** → can query and continue the work

---

## Specification

Artyfacts implements the **Agent Artifact Handoff (AAH)** specification — an open standard for agent-produced artifacts.

- [AAH Specification v0.1](./AAH-SPEC-v0.1.md) — The artifact format
- [Product Specification](./PRODUCT-SPEC.md) — The storage + viewer layer
- [Data Model](./DATA-MODEL.md) — Database schema with versioning + dedup

---

## Integrations

| Framework | Status |
|-----------|--------|
| [Clawdbot](https://github.com/clawdbot/clawdbot) | 🔜 Planned |
| [LangChain](https://langchain.com) | 🔜 Planned |
| [CrewAI](https://crewai.com) | 🔜 Planned |
| [AutoGen](https://microsoft.github.io/autogen/) | 🔜 Planned |
| Custom (REST API) | ✅ Day 1 |

---

## Pricing

| Tier | Price | Artifacts | Retention |
|------|-------|-----------|-----------|
| **Free** | $0/mo | 100/mo | 7 days |
| **Pro** | $20/mo | 2,000/mo | 90 days |
| **Team** | $50/mo | 10,000/mo | 1 year |
| **Enterprise** | Custom | Unlimited | Custom |

---

## Status

🚧 **Pre-launch** — We're building this right now.

Interested? Star this repo and watch for updates.

---

## About

Artyfacts is built by [Artygroup](https://artygroup.dev).

*Because your agents deserve better than a messy folder of markdown files.*
