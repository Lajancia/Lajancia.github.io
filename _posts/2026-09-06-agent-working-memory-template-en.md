---
title: "A Working Memory Vault for AI Coding Agents — Design and Practice of the Agent Working Memory Template"
date: 2026-09-06 12:00:00 +0900
categories: [Study]
tags: [AI, Agent, LLM, Claude, Codex, Working Memory, Obsidian, Template]
author: L.J
lang: en
---

### **Introduction**

AI coding agents are evolving fast. Claude now has Memory and session search. Context windows keep growing. Every generation is more capable than the last.

But progress creates a new problem.

**Your knowledge becomes agent-dependent.**

What you built up in Claude stays in Claude. Work done in Codex is locked inside Codex sessions. When a better agent comes along tomorrow — and it will — you start over. Re-explain your stack, your preferences, your decisions, your recurring patterns, from scratch.

This template exists for one reason:

> **To centralize your work knowledge in an agent-agnostic way, so no matter which agent you use — today or in the future — connecting the vault gives you immediate continuity.**

Agents are tools. Tools change. But the knowledge and context you've accumulated should outlive any single tool.

---

### **What This Solves**

#### **1. Agent Lock-In**

Claude Memory is useful inside Claude. Session search works within the same agent. Switch to Codex? Move to Cursor? Upgrade to the next-generation agent?

**Knowledge locked inside a specific tool makes migrating to better tools expensive.**

| Situation | Problem |
|-----------|---------|
| Claude Memory | Convenient inside Claude, but no other agent can access it |
| Session search | Sessions don't evaporate, but they're unstructured chat dumps |
| Agent migration | Every new agent requires re-explaining context from scratch |

#### **2. Context Fragmentation**

The more agents you use in parallel, the more fragmented your context becomes. "This decision was made in Claude, that code was written in Codex..." — everything scattered across different sessions.

#### **3. Unstructured Information**

Even with session search, chat logs are fundamentally unstructured. Decisions, work patterns, and reusable workflows are buried inside message streams, making them hard to reuse without manual extraction.

---

### **Design Principles**

| Principle | Core Idea |
|-----------|-----------|
| **Agent-Agnostic** | The vault is not tied to any specific agent. Claude, Codex, or any future agent can read and write the same vault |
| **Centralized** | Context from every session converges in one place. No matter which agent you used, the information stays in the vault |
| **Personal vault first** | Optimized around one person's work style, project status, and preferences. Not designed for multi-user sharing |
| **Start with minimal routine** | Morning Briefing and Wrap-up are the default triggers. Tune timing and scope through actual use |
| **Explicit sources & uncertainty** | The agent distinguishes files it actually checked from external sources it couldn't access. Unqueried systems are marked "not queried" |
| **Tiered recording levels** | Must / Should / Maybe / Do not write — four levels to maintain signal-to-noise ratio |
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

### **Relation to Claude Memory / Session Search**

This template does **not replace** Claude's Memory or session search. It complements them.

| Feature | Strength | Limitation |
|---------|----------|------------|
| **Claude Memory** | Fast in-agent reference, auto-learning | Claude-only. No export/portability. Unstructured |
| **Session Search** | Find past conversations | Unstructured data. Hard to extract decisions/patterns. Agent-dependent |
| **This Vault** | Agent-agnostic. Structured knowledge. Fully portable | Requires initial setup. Needs routine maintenance |

Using all three together is the most effective approach. Claude Memory for quick in-session reference, this vault for structured long-term preservation, and any agent can connect to it when needed.

---

### **Referenced Patterns**

| Pattern | Core Idea | Reflection in Template |
|---------|-----------|----------------------|
| **Karpathy LLM Wiki** | A file-based wiki LLMs can read, organize, and connect over time | Projects, meetings, people, tools, and concepts are managed as Markdown wiki |
| **OpenClaw Auto-Dream** | Periodically summarize session history and promote important info to long-term memory | Simplified into Journal → Observations → Working_Style flow |
| **Daily Review / Shutdown** | Use daily reviews and shutdown rituals to define start and end of day | Morning Briefing and Wrap-up form the minimum refresh cycle |

---

### **Expected Benefits**

- **Use any agent** — connect the same vault and start working immediately
- **Free migration** — move from Claude to Codex to a future better agent without losing context
- **Decisions and corrections survive agent changes**
- **Centralized context** — even when using multiple agents in parallel
- **Knowledge scattered across chats accumulates into a personal work wiki**
- Claude Memory + vault = **short-term reference + long-term preservation** optimized
- Morning Briefing and Wrap-up keep the vault from going stale
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

### **Real-World Experience — It's Not Perfect, and That's OK**

The most important lesson from actually using this template: **Perfect automation doesn't exist.**

Despite defining Morning Briefing and Wrap-up routines, agents sometimes skip them. The vault is the last layer an agent references — when the agent is deep in work, it's easy to miss the briefing. So there are days when I still need to manually trigger "morning briefing" or "wrap up today."

This isn't a limitation of the template. **It's the reality of any knowledge management tool.**

| Reality | Response |
|---------|----------|
| Agent occasionally skips Morning Briefing | Manually trigger "morning briefing" — takes one second |
| Session ends without Wrap-up | `hot.md` preserves last-known state; next session recovers from it |
| Recording depth is inconsistent | Tweak Working_Style.md to adjust the threshold |
| Vault gets cluttered | Periodically promote patterns to `Observations.md` and archive old journals |

The key insight: **don't try to craft perfect rules upfront. Use the template, notice friction points, and continuously reflect improvements in `Working_Style.md` and `AGENTS.md`.**

---

### **Closing Thoughts — The Vault Outlives the Agent**

Claude will keep evolving. Better agents will keep appearing. That doesn't diminish the value of this template — it amplifies it.

**The better agents get, the more important it becomes to keep your knowledge independent of any single one.**

If tomorrow's superior agent has no idea about the context you built in Claude today, it will have to learn everything about you from scratch. This vault bridges that gap. No matter which agent you use — today or five years from now — connecting the vault restores your work context instantly.

It's not a silver bullet, but this is how I use AI agents in my work. Acknowledging that it's not perfect is what makes it sustainable in the long run.

> **GitHub**: [github.com/Lajancia/agent-working-memory-template](https://github.com/Lajancia/agent-working-memory-template)