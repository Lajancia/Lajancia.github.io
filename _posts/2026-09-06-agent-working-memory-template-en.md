---
title: "A Working Memory Vault for AI Coding Agents — Design and Practice of the Agent Working Memory Template"
date: 2026-09-06 12:00:00 +0900
categories: [Study]
tags: [AI, Agent, LLM, Claude, Codex, Working Memory, Obsidian, Template]
author: L.J
lang: en
---

### **Introduction**

If you use AI coding agents like Claude, Codex, or Cursor in your daily workflow, you've probably noticed a common frustration: **every session starts from scratch**.

- "This project uses pnpm, not npm."
- "Please respond in English, and keep it concise."
- "We decided on this architecture last week."

You end up repeating yourself session after session. AI coding agents have **ephemeral sessions** by default — once a session ends, the context is lost. Different tools (Codex, Claude) each maintain their own chat history, making it hard to move between them seamlessly.

**Agent Working Memory Template** was built to solve this. It's a file-based personal working memory vault that gives your AI agents session continuity and lets them accumulate work knowledge automatically.

---

### **Problems Addressed**

| Problem | Symptom |
|---------|---------|
| **Session ephemerality** | When tools change or sessions reset, prior context is lost. You repeatedly explain current state, preferred style, and recording boundaries. |
| **Disconnection between agents** | Codex (code) and Claude (docs/long-context) have different strengths, but each has its own chat history — requiring manual handoffs every time. |
| **Non-accumulating knowledge** | Meeting notes, decisions, corrections, and recurring solution patterns live only in chat and never become a reusable asset. |

---

### **Design Principles**

| Principle | Core Idea |
|-----------|-----------|
| **Personal vault first** | Optimized around one person's work style, project status, and preferences. Not designed for multi-user sharing. |
| **Start with minimal routine** | Morning Briefing and Wrap-up are the default triggers. Tune timing and scope through actual use. |
| **Explicit sources & uncertainty** | The agent distinguishes files it actually checked from external sources it couldn't access. Unqueried systems are marked "not queried." |
| **Tiered recording levels** | Must / Should / Maybe / Do not write — four levels to maintain signal-to-noise ratio. |
| **Bootability** | `hot.md` serves as a boot cache answering: "where are we now, and what should happen next?" |

---

### **Overall Structure**

The template consists of three layers.

```
Agent_Working_Memory_Template/
├─ AGENTS.md                         # Operating rules for the agent
├─ README.md                         # Quick-start guide for users
├─ SETUP.md                          # Setup questions and initialization guide
├─ hot.md                            # Boot cache for the next session
├─ log.md                            # Vault operation log (append-only)
├─ 00_Index.md                       # Vault index
├─ 01_Daily/                         # Daily notes and work logs
├─ 10_Projects/                      # Project wiki
├─ 20_Meetings/                      # Meeting notes and action items
├─ 30_Agent/
│  ├─ Working_Style.md               # User preferences and execution rules
│  ├─ Observations.md                # Recurring observed patterns
│  └─ Journal.md                     # Latest 1-2 session summaries
├─ 70_Entities/                      # People, tools, and concepts
│  ├─ People/
│  ├─ Tools/
│  └─ Concepts/
├─ 90_Retrospectives/                # Daily/session retrospectives
└─ 99_Templates/                     # Note templates
```

**Three layers and their roles:**

| Layer | Role | Representative Files |
|-------|------|---------------------|
| **Agent Continuity** | Restore the same context and style across sessions | `hot.md`, `Working_Style.md`, `Observations.md` |
| **Autonomous Wiki Growth** | Accumulate work knowledge into a personal wiki | `10_Projects/`, `20_Meetings/`, `70_Entities/` |
| **Routine-Based Refresh** | Prevent the vault from going stale | `01_Daily/`, `90_Retrospectives/`, `Journal.md` |

---

### **Core Routines**

**1. Setup (First-time configuration)**

When the user says "Use this folder as my working memory vault," the agent reads `AGENTS.md` and `SETUP.md`, asks 3–6 questions, and initializes the vault with user info, projects, briefing sources, and recording boundaries.

**2. Morning Briefing**

```text
Please give me a morning briefing from this working memory vault.
```

The agent reads `hot.md`, the latest daily note, project index, and configured sources to summarize:
- Today's focus
- Pending items
- Blockers
- External inputs
- Independent work

If today's daily note doesn't exist, it's auto-created from `99_Templates/daily.md`.

**3. During Work (Automatic recording)**

The agent selectively records only what matters:
- **Must write** — user corrections, explicit decisions, handoffs, external system status changes, meeting/document ingest results
- **Should write** — recurring work patterns, reusable workflows, bug cause/fix patterns
- **Maybe write** — brainstorming, one-off opinions, unverified interpretations
- **Do not write** — simple conversational reactions, duplicates already in `hot.md`, unverified guesses, sensitive information

**4. Wrap-Up**

```text
Please wrap up today.
```

The agent updates the daily note, creates/updates a retrospective, rewrites `hot.md` as a boot cache, and adds summaries to `Journal.md` and `log.md`. Recurring patterns are promoted to `Observations.md` or project notes.

---

### **Information Promotion Flow**

```
Conversation
    → Daily note / Journal
    → hot.md
    → Project / Meeting / Entity notes
    → Observations
    → Working_Style
```

| Stage | Role |
|-------|------|
| **Conversation** | Raw context needed for immediate work |
| **Daily note / Journal** | Compressed record of the day or session |
| **hot.md** | Short boot cache for the next session |
| **Project / Meeting / Entity notes** | Long-term work knowledge |
| **Observations** | Recurring observed patterns |
| **Working_Style** | Preferences validated enough to become execution rules |

This flow prevents one-off information from becoming a permanent user rule too quickly.

---

### **Referenced Patterns**

| Pattern | Core Idea | Reflection in Template |
|---------|-----------|----------------------|
| **Karpathy LLM Wiki** | A file-based wiki LLMs can read, organize, and connect over time | Projects, meetings, people, tools, and concepts are managed as Markdown wiki |
| **OpenClaw Auto-Dream** | Periodically summarize session history and promote important info to long-term memory | Simplified into Journal → Observations → Working_Style flow |
| **Daily Review / Shutdown** | Use daily reviews and shutdown rituals to define start and end of day | Morning Briefing and Wrap-up form the minimum refresh cycle |

---

### **Expected Benefits**

- Less time spent restoring context after a session reset
- Move between Claude and Codex while using the same context source
- Decisions and corrections from work are never lost
- The agent gradually personalizes around your work style
- Knowledge scattered in chat accumulates into a personal work wiki
- Morning Briefing and Wrap-up keep the vault from going stale
- Checked sources and unavailable sources are clearly separated
- A human-readable file structure anyone can inspect, edit, and correct directly

---

### **Limitations and Recommended Use**

This template is a **personal working memory vault** shaped around one person's work style. It is **not recommended as a shared team knowledge base**.

- Each person copies the template and uses it as a personal vault
- The personal vault stores work style, current state, handoffs, and observations
- Team-level decisions or project knowledge should be promoted to a separate team wiki or issue tracker
- Sensitive information is not automatically recorded without explicit approval
- Update timing and recording scope are adjusted through actual use

---

### **Useful Prompt Examples**

| Intent | Prompt |
|--------|--------|
| Initial setup | "Use this folder as my working memory vault. Please run setup." |
| Start of day | "Please give me a morning briefing." |
| End of day | "Please wrap up today." |
| Meeting ingest | "Please turn this meeting transcript into meeting notes and update the vault." |
| Reflect a correction | "The corrected schedule is important for the next session, so please reflect it in hot.md and the daily note." |
| Exclude from record | "This is personal, so please do not record it in the vault." |

---

### **Closing Thoughts**

The Agent Working Memory Template is a practical tool for addressing the biggest weakness of AI coding agents: **session volatility**. Without complex infrastructure or external services, a simple set of Markdown files and an Obsidian vault structure can give AI agents persistent context and self-accumulating knowledge.

The guiding philosophy: **"Rather than making perfect rules from the start, refine `Working_Style.md` and `AGENTS.md` based on what feels awkward during actual use."**

> **GitHub**: [github.com/Lajancia/agent-working-memory-template](https://github.com/Lajancia/agent-working-memory-template)