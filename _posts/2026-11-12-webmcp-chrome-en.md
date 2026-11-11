---
title: "A Retrospective on Google Chrome's WebMCP: How a Web Page Hands Tools to an AI Agent, and What I Learned Building One"
description: "Not a spec review. A field report on what works and what doesn't yet."
date: 2026-11-12 00:40:00 +0900
categories: [Development]
tags: [WebMCP, MCP, Google Chrome, AI Agent, Next.js, Frontend]
author: L.J
lang: en
---

*On November 7, at BiznagaFest 2026 in Málaga, I first ran into WebMCP, the proposal led by the Google Chrome team. Reading about it wasn't enough, so I built a small experiment to try the spec for real: a page that an agent fills in like a newspaper with my schedule and work. The result is on GitHub. The newspaper is just a test vehicle; the subject of this post is WebMCP itself. I start from the basics so that even if you've never heard of WebMCP you can follow along, then look back on what I learned and how far it works today.*

> 이 글의 한국어 버전은 [여기](/posts/webmcp-chrome-ko/)에 있습니다.

## 1. WebMCP in plain words

### Start with MCP

For an AI agent (an AI that searches, books, and schedules things on your behalf) to actually get work done, it has to use features of outside services. MCP (Model Context Protocol) is a shared standard for that connection. A service publishes a list of what it can do, such as "search flights" or "add a calendar event" (these are called tools), and the agent reads the list and calls what it needs. It's similar to how phones converged on USB-C so that any charger fits.

In classic MCP, the party publishing that tool list is a server (the backend). That's where the friction comes from. The agent talks directly to the server and knows nothing about the screen you're looking at in your browser. Developers also have to copy login state and page state onto the server separately.

### WebMCP changes who publishes the tools

At BiznagaFest, Raúl Tovar (The Workshop) introduced WebMCP in his session "Construyendo la Web Agéntica" ("Building the Agentic Web"). In one line:

> **With MCP, a server gives tools to an agent. With WebMCP, a web page does.**

Think of a restaurant. MCP is like phoning your order in to the kitchen: it works, but the kitchen can't see what you're looking at at your table. WebMCP is the tablet menu on the table. You and the agent look at the same screen, and orders go through the buttons (tools) that the screen provides.

### It also differs from how agents use the web today

Browser agents usually read the screen (HTML or a screenshot) and click like a person would. If a button moves or gets renamed, things break. WebMCP is the page handing over an official instruction manual: "don't click around, just call this function."

Four differences at a glance:

- **Who registers the tools:** MCP uses a backend server; WebMCP uses the web page inside the browser.
- **Where the agent connects:** MCP goes through an API server, bypassing the page. WebMCP goes through the very tab the user has open.
- **State and auth:** with MCP you have to duplicate them on a server. With WebMCP the page already has them and just uses them.
- **The screen (UI):** with MCP the agent can't see it. With WebMCP the user and the agent look at the same screen (shared context).

### What it looks like in code

The page registers a tool in JavaScript. Trimmed down from what I actually used:

```js
// Newest location is document; fall back to navigator on older Chrome
const mc = document.modelContext ?? navigator.modelContext;

mc?.registerTool({
  name: "add_headline",                       // the tool's name
  description: "Add one main headline to the newspaper", // what the agent reads
  inputSchema: {                              // the shape of accepted arguments
    type: "object",
    properties: { title: { type: "string" }, url: { type: "string" } },
  },
  execute: async (args) => { render(args); return { ok: true }; }, // what actually happens
});
```

Reading it is easy. `name` and `description` are the tool's label and manual. `inputSchema` fixes the shape of the values it accepts ("title is a string, link is a string"), so the agent can't make up parameters. `execute` is what the page actually does when the tool is called. Instead of poking around the DOM, the agent just reads this "contract" and calls `add_headline` with a title and a link. All the code that draws the screen lives inside the page, so as long as the tool signature stays the same, a layout redesign won't break the call.

### Where it stands today

The spec is a W3C Community Group draft, and the GitHub repo (webmachinelearning/webmcp) lists engineers from Google Chrome and Microsoft as co-authors. It began as a flag-gated preview in Chrome 146 Canary, and an origin trial (a test period in which a site can request a token and turn the feature on early) opened in Chrome 149. It is not a finished standard.

## 2. Where it fits, and the trade-offs

### How a call flows

1. The user opens a page that supports WebMCP.
2. The agent discovers the tools registered in that tab (discovery).
3. The agent calls a tool with arguments.
4. The page runs `execute()` and updates the screen. The browser mediates the whole exchange.

**Good fits:** complex forms with split fields (first name, last name, address), multi-leg booking flows, diagnostic triggers buried deep in settings pages.

### Pros

- You get an agent touchpoint with client-side JS only, no backend.
- Page state and agent actions happen in the same tab, so context isn't torn apart.
- Unlike DOM scraping, it survives layout changes.

### Cons

- **Tool lifetime is tied to the tab.** Tools exist only while the tab is open, so background batch jobs are out of scope.
- **The client ecosystem is young.** Few agents act as WebMCP clients. The page side is ready, but almost nothing consumes it. That's effectively the biggest bottleneck.
- **The spec is still moving.** The July 2026 draft moved the API from `navigator.modelContext` to `document.modelContext`, and Chrome 150 deprecated the old location. Checking both is now a must.
- **There's a refactoring cost.** To expose something as a tool, you have to pull render logic out of components and into `execute()`. In an existing React app, that's real work.

## 3. The experiment: a newspaper the agent fills in

To try WebMCP hands-on, I picked one idea: **a page where an agent collects my schedule, work, and news interests every day and lays them out like a front page.** It was just the right size to test whether a single page, with no server, could become an agent touchpoint.

I set one design rule: **serverless.** No backend and no server-side user data. News and schedule are a one-off stream the agent collects at fill time, and a refresh gives you a blank page. Data lives only in the memory of the open tab. Deployment is just static file hosting, so each visitor's own agent fills their own paper in their own browser. With no shared server state, every user gets a fully independent instance, and multi-user support fell out of the architecture as a side effect.

The stack is Next.js (App Router) + Tailwind + shadcn/ui, with five tools: `set_date`, `add_headline`, `add_story`, `add_task`, and `clear_paper`.

## 4. What I learned building it

### (1) Define tools in one place

I split tool definitions (name, schema, execute), the state store, and the UI components into three layers, so I can tear up the newspaper layout later without touching the contract. All rendering lives inside `execute()`. The key is never exposing a DOM API to the agent.

### (2) The path that works today: a mirrored registry

Native WebMCP clients are rare, so the page does two things at once:

- It registers properly on `document.modelContext` (or `navigator.modelContext` on older builds). In supporting browsers it switches to the native path immediately.
- It mirrors the same tool objects onto `window.__NEWSPAPER_TOOLS__`. An agent attaches to the tab over CDP (Chrome DevTools Protocol, the channel developer tools use to remote-control a browser) and runs `tool.execute(args)` inside the tab. It's a pipe that passes the same contract through today, without browser mediation.

```
agent ──CDP──▶ window.__NEWSPAPER_TOOLS__ in the tab ──▶ execute(args) ──▶ render
        (today)

agent ──WebMCP client──▶ document.modelContext ──▶ execute(args) ──▶ render
        (later)       ↑ only this pipe changes; the contract stays
```

### (3) Security: agent arguments aren't trustworthy either

Even a value sent by my own agent becomes a vulnerability if it's drawn on screen without checking. Here's what I confirmed with attack-style test inputs:

- A link starting with `javascript:` arriving as a news URL runs malicious script on one click (XSS). I blocked it with a scheme allowlist that permits only `http/https`.
- Control characters (newlines, tabs) break headline layout. I strip them and cap each field's length.
- If the bridge picks a tab with prefix matching like `url.startsWith()`, a sibling path (`<url>-evil`) can hijack the tool channel. Compare origin + pathname strictly.
- When testing over a debug port, never use your main Chrome profile. Use a dedicated `--user-data-dir`.

In short: **sanitize at the page boundary, match strictly at the bridge boundary, and verify tests against DOM state.** A tool can return `{ok:true}` and you still have to read the actually rendered DOM to know.

### (4) Design the transition up front

Moving from bridge to native only swaps the pipe. The flow (discover tools, collect data, call execute) is identical. If Chrome standardizes this and agents become native clients, only the middleman gets replaced. It's a "ship the contract first, let the pipe arrive later" design.

## 5. Current limitations, honestly

- **Every fill so far went through the workaround path.** Native WebMCP clients aren't mainstream yet, so I used a pipe that attaches over CDP, skipping the browser's consent UI. That differs from the standard's intended browser-mediated model: the page-side contract is real WebMCP, but the client-side pipe is not yet. I designed it as a transition path, not a failure, but I want that distinction to be clear.
- **Tools aren't permanent.** Close the tab and the tools vanish. Recurring background work has to wait until the Service Worker integration discussion matures.
- **No cross-device sync.** I don't use localStorage by default either, so with multiple devices you get independent papers. That's the price of leaving out a server, and I chose to pay it.

## 6. Looking back: takeaways

- **MCP is a server handing out tools; WebMCP is a page handing them out.** Instead of agents scraping the web, the page gives agents an official contract.
- The page side can ship today. A few dozen lines of registration code establish the contract, and layout changes won't break it.
- The real bottleneck isn't pages but the **client ecosystem.** The standard takes hold only when more agents consume WebMCP.
- Agent arguments are an attack surface too. Sanitizing at the page boundary is new work for frontend developers.

I don't know yet whether this standard, pushed by Google Chrome, will take hold. But the direction, pages handing agents an official contract, is persuasive. My small newspaper was an experiment that showed both the promise and the limits. The agent fills the paper, and I read it. The server hosting the page still knows nothing.

*Repo: [github.com/Lajancia/newspaper-web-mcp](https://github.com/Lajancia/newspaper-web-mcp) (Next.js + shadcn/ui, includes tool registration / bridge / sanitize)*
