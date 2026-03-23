---
layout: post
title: "Claude AI: The Features You're Probably Not Using (But Should Be)"
date: 2026-03-23
description: "A practical tour of Claude's most powerful and overlooked capabilities — Artifacts, Projects, MCP, prompt engineering patterns, and the tricks that actually change how you work."
tags:
  - "Claude AI"
  - "AI tools"
  - "prompt engineering"
  - "productivity"
  - "MCP"
giscus_comments: true
---

Most people use Claude like a slightly smarter search engine. Type a question, get an answer, close the tab. That's leaving an enormous amount on the table. Claude has grown into a layered platform with agentic tooling, persistent workspaces, and an interactive runtime — and the gap between a casual user and someone who's actually configured it properly is massive. Here's the tour I wish I'd had.

{%- include figure.html path="assets/img/blog/claude-ai-the-features-youre-probably-not-using-bu/03_illustration.jpg" alt="Claude AI split-panel interface showing live artifact rendering alongside a chat conversation" class="img-fluid rounded z-depth-1" -%}

## First: Pick the Right Model

This sounds obvious until you realize most people just use whatever the default is and call it a day.

Claude comes in three tiers — **Haiku**, **Sonnet**, and **Opus** — and matching the model to the task is itself a productivity hack. Haiku is fast and cheap, great for high-volume classification, summarisation, or anything where latency matters. Sonnet is the daily driver: balanced, capable, handles most writing and coding tasks without drama. Opus is for the hard stuff — complex reasoning, long document synthesis, anything where you'd rather wait a few extra seconds and get it right.

Running everything through Opus because it sounds most impressive is like using a sledgehammer to hang a picture frame. Pick Haiku when speed matters, Sonnet when you just want to get work done, and Opus when the task genuinely warrants it.

## Artifacts: The Hidden Interactive Runtime

This is the feature that changes everything and somehow most people walk right past it.

Artifacts is a side panel that renders live, interactive output — HTML pages, SVG graphics, React components, dashboards, mini-apps, even games — directly inside the chat. You're not just reading generated code; you're running it. Ask Claude to build a calculator, a data visualisation, or a resume builder, and it appears on the right, live and clickable.

The stack it supports out of the box is genuinely impressive: React with Tailwind CSS, Shadcn UI, Lucide icons, and Recharts. A single prompt can produce a fully styled, interactive component.

A prompt that actually works:

```
Build a React component using Tailwind CSS that shows a kanban board 
with three columns (To Do, In Progress, Done) and drag-and-drop cards. 
Each card should have a title, tag colour, and due date. 
Make it functional with local state — no external libraries beyond what's available.
```

You get a working kanban board. In the chat. Without spinning up a dev environment.

A few things worth knowing:

- Adding **"using AI"** to your Artifact prompt tells Claude to embed reasoning logic into the output — making it smarter, not just prettier.
- Artifact **version history** works like a lightweight Git: you can inspect and revert to any previous iteration if Claude goes off in the wrong direction.
- **Custom Visuals** (a newer addition) lets you request bespoke flowcharts, comparison tables, and diagrams — Claude generates them as proper SVG rather than ASCII art.

> **Tip:** If you want to share an Artifact externally, Claude can publish it to a shareable URL. Great for quick prototypes or internal demos where spinning up a Vercel deployment would be overkill.

## Projects: Give Claude a Permanent Memory

{%- include figure.html path="assets/img/blog/claude-ai-the-features-youre-probably-not-using-bu/04_illustration.png" alt="Claude Projects interface showing the &#x27;Set custom instructions for project&#x27; dialog" class="img-fluid rounded z-depth-1" -%}

The biggest friction point with AI assistants is the amnesia. Every session starts from scratch — you re-explain the project, re-paste the context, re-specify the tone. Projects fix this.

A Project is a persistent workspace that retains:
- Your uploaded files (PDFs, spreadsheets, code, images) as a searchable **Knowledge Base**
- **Custom instructions** — a permanent system prompt that shapes how Claude responds in that project
- **Full conversation history** across every session

I have a Project for every non-trivial ongoing context: one for a research area I'm tracking, one for a codebase I maintain, one for writing work with a specific style guide. Claude walks into each session already briefed.

Here's what a useful custom instruction looks like:

```
You are assisting with an open-source Rust library. 
The codebase targets stable Rust, no nightly features. 
Always suggest idiomatic Rust — prefer iterators over explicit loops, 
use proper error propagation with `?`, and avoid `.unwrap()` in library code. 
When in doubt, ask a clarifying question before generating code.
```

Set that once. Never type it again.

> **Tip:** If you're working on multiple related files, upload them all to the Project Knowledge Base. Claude can cross-reference between documents — useful for things like "does this API design contradict what we documented in the spec last month?"

Claude's native **Memory** feature (plan-dependent) goes a step further and persists preferences *across* projects. Combined with a file-based memory system using a local MCP server, you can get cross-session recall that works simultaneously in Claude.ai, Claude Code, and the API.

## Prompt Engineering That Actually Works

There are a few patterns worth internalising. Not because they're magic, but because they cut rewrites.

**The Role → Context → Task → Format → Constraints pattern:**

```
Role: You are a senior backend engineer reviewing code for production readiness.
Context: This is a Node.js API endpoint that handles payment webhooks from Stripe.
Task: Review the code below and identify issues with error handling, idempotency, and security.
Format: Bullet points grouped by severity — Critical, Warning, Info.
Constraints: Be direct. If something is fine, say so. Don't pad the output.
```

Structuring prompts this way turns Claude from a chatbot into a structured thinking tool. The output is predictable and usually needs one revision rather than three.

**Reverse prompting** is the underrated one. Instead of jumping straight to output, ask Claude to interview you first:

```
I want to write a technical post-mortem for a production outage. 
Before you draft anything, ask me the questions you need answered 
to write a thorough and useful post-mortem.
```

This surfaces requirements you didn't know you had. I use it constantly for anything where scoping matters more than speed.

**Constraint cascades** — iteratively layering constraints rather than front-loading them — work surprisingly well for refining outputs. Start broad, get a draft, then add: *"Now tighten it to 400 words," "Now make the tone more direct," "Now add a concrete example in the second paragraph."* Each constraint sharpens without losing the thread.

> **Tip:** Claude's **Custom Styles** feature (in the UI: Normal / Explanatory / Formal / Custom) applies your preferred tone across every response without you specifying it per prompt. Set it once for your account.

**Verification loops** are what Anthropic engineers apparently use internally — asking Claude to check its own answer before returning it:

```
... Before responding, re-read your answer and verify that:
1. The code compiles without errors
2. You haven't introduced any off-by-one errors
3. The approach handles the empty list case
If any check fails, fix it first.
```

Anecdotally, this reduces obvious errors by a meaningful amount on tasks where correctness is load-bearing.

## MCP and Connectors: Where It Gets Agentic

Claude **Connectors** are the consumer-facing integration layer: link Claude to your email, calendar, Google Drive, and external databases so it answers questions grounded in your actual context, not generic training data.

**MCP (Model Context Protocol)** is the developer-facing version of the same idea — a standardised protocol that lets Claude connect to thousands of external tools via a common interface. Live databases, filesystems, Supabase, GitHub, web scrapers — all accessible from chat or Claude Code without custom glue code.

The **Advanced Tool Use API** (currently in beta) pushes this further with two features that matter for complex workflows:

- **Tool Search Tool** — Claude dynamically finds the right tool from a large registry without stuffing the entire toolset into the context window
- **Programmatic Tool Calling** — Claude writes code to orchestrate multiple tools in a single pass, collapsing multi-step pipelines into one request

Combine Projects + MCP + the 200K token context window on Opus, and you can drop an entire codebase or legal document set into a persistent workspace and have Claude reason across it indefinitely. That's a qualitatively different capability from what most people associate with "chat AI."

> **Tip:** Start with one MCP server, not ten. The [Context7 MCP](https://github.com/upstash/context7) is a good first one — it fetches current library documentation on demand, so Claude stops hallucinating outdated API signatures.

## The Payoff

The tooling has matured enough that the bottleneck is no longer what Claude can do — it's whether you've configured the environment to let it. That setup investment pays back quickly. An afternoon spent on Projects, a prompt pattern you trust, and one MCP integration that hits a real data source you use daily will change how you work with it. Start there.