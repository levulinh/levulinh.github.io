---
layout: post
title: "I Built a Virtual Software Company for $100/Month"
date: 2026-04-04
description: "Five Claude agents acting as PM, Designer, Programmer, QA, and CTO — coordinating through Linear and GitHub to ship a real Flutter app. Here's how I built it, what broke, and what surprised me."
tags:
  - "AI"
  - "agentic AI"
  - "Claude"
  - "software engineering"
  - "automation"
  - "MTBox"
giscus_comments: true
---

I've always wanted to build a software product under the name MTBox — a name I really resonate with. But I kept dropping projects for the same reason every time: I'd still have to sit in front of a computer, wait for the agent to do its job, check, prompt again. Boring. And the results were always unsatisfying, because I was always too ambitious and asked too much from a single agent. You know — like your bosses do in real life.

So I stopped trying to give one agent a 100-point todo list. Instead, I asked: what if I organized them like a company?

{%- include figure.html path="assets/img/blog/mtbox-virtual-company/dashboard.png" alt="The MTBox agent dashboard — five agents running autonomously on a polling schedule" class="img-fluid rounded z-depth-1" -%}

## The Setup: An Org Chart Made of Shell Scripts

MTBox is not a product. It's the orchestration infrastructure for an AI-powered software company. Five Claude Code agents — each with a defined role, a prompt file, and a run script — collaborate to build an actual Flutter app without me sitting in front of a terminal.

The cast:

- **PM** — reads Linear, triages backlog, moves issues through workflow statuses, writes acceptance criteria
- **Designer** — picks up "In Design" issues, generates Flutter mockups, posts them to Linear for review
- **Programmer** — takes approved designs, writes the actual Dart/Flutter code, opens a PR
- **QA** — clones the PR branch, runs unit and widget tests, reports pass/fail back to Linear
- **CTO** — reads high-level directives from me (the "CEO"), creates roadmap phases, and seeds tasks for the PM to pick up

Each agent runs on a two-hour polling interval via `launchd`. No manual prompting required. I fire up my Mac in the morning, and by evening the company has been running shifts.

The real product — a Flutter app called **Campaign Tracker** — lives in a separate repo. MTBox is just the scaffolding.

## Coordination Without a Chat Room

The key insight was: agents don't need to talk to each other directly. They need a shared source of truth.

That source of truth is **Linear** — specifically, the issue status transitions.

```
Backlog → In Design → Awaiting Design Approval
→ In Progress → In Review → Awaiting Decision → Done
```

Only the PM moves issues between statuses. Everyone else reads the current state, does their job, posts a comment, and leaves the next move to PM. It's like a relay race where the baton is an issue ticket.

{%- include figure.html path="assets/img/blog/mtbox-virtual-company/linear-board.png" alt="Linear issue board showing Campaign Tracker tasks across all workflow stages" class="img-fluid rounded z-depth-1" -%}

Each agent identifies itself in comments with a prefix: `📋 [PM]`, `🎨 [Designer]`, `💻 [Programmer]`, `🔍 [QA]`, `🏢 [CTO]`. This makes the activity feed readable as a conversation — which it effectively is.

## Watching a Feature Ship

Here's what it looks like in practice. I gave the CTO a directive: *"Add some touches so the app doesn't feel too boring."* The CTO translated that into a Phase 3 roadmap item — "Visual delight polish: micro-interactions, empty states, subtle transitions" — and dropped it in the backlog.

PM picked it up, wrote acceptance criteria, and moved it to "In Design."

Designer opened the issue, built a mockup in Flutter, and posted it with detailed panel-by-panel specs:

{%- include figure.html path="assets/img/blog/mtbox-virtual-company/linear-issue-comments.png" alt="Designer Bot posting a Flutter mockup with design notes for the Visual Delight Polish issue" class="img-fluid rounded z-depth-1" -%}

Then CTO reviewed the design and either approved or pushed back. In this case: approved, with notes on which acceptance criteria were satisfied. Programmer picked it up, implemented it, and opened a PR. QA ran 20 unit and widget tests — all passed.

{%- include figure.html path="assets/img/blog/mtbox-virtual-company/linear-issue-comments2.png" alt="Programmer Bot reporting implementation complete, QA Bot reporting 20/20 tests passing" class="img-fluid rounded z-depth-1" -%}

The whole thing — from directive to green tests — happened while I wasn't watching. That's the part that still surprises me every time I check in.

## The Evolution: What I Built and Then Changed

The first version was embarrassingly simple: five shell scripts, five prompt files, a `launchd` plist to run each every 15 minutes. That's it. No dashboard, no status tracking, no coordination layer beyond Linear itself.

**Round 1 — Basic agents + polling.** PM, Designer, Programmer, QA. Each runs on a cron-like schedule, checks Linear for work in their lane, does the work, posts a comment. Lock files prevent two instances of the same agent running simultaneously.

**Round 2 — The dashboard.** Watching logs with `tail -f` is fine for 10 minutes. For hours of autonomous operation, I needed something better. I built a Node.js server with a brutalist-aesthetic HTML dashboard: five agent cards, IDLE/BUSY status, countdown timer to next run, a live log stream. The dashboard became how I "check in" without micromanaging.

**Round 3 — The CTO agent.** The original setup required me to manually seed tasks. The CTO abstraction gave me a layer between my high-level directions ("build a campaign tracker app") and the PM's day-to-day task management. I write directives; CTO translates them into phased roadmaps; PM executes. Now I genuinely only need to show up when something needs a decision.

**Round 4 — Linear webhooks.** Polling every two hours is fine for overnight runs, but during active development it creates 30-120 minute lags between one agent finishing and the next picking up. Adding a Cloudflare quick tunnel + Linear webhook integration meant that when an issue changes status, the right agent fires immediately. The polling schedule stays as a safety net; the webhooks handle real-time handoffs.

**Round 5 — Cost optimization.** Once this was actually running, I noticed that PM and QA don't need Sonnet-level intelligence for their jobs — reading Linear, moving issues, running tests. Switching them to Haiku cut token usage significantly. Extending polling intervals from 15 minutes to 2 hours on all agents was the other big lever. The agents were mostly running idle checks anyway.

**Round 6 — Pause/resume.** Sometimes I want to push code changes or adjust prompts without the agents firing mid-edit. A PAUSE button in the dashboard with an auto-resume timer was the last quality-of-life feature. Small thing, but it made the system feel more like a tool I control rather than a process I'm watching nervously.

## What Actually Got Built

Campaign Tracker is a Flutter app for tracking personal goals with a streak mechanic. In one day of running, the company completed:

**Phase 1** — Core check-in loop: app shell, campaign list, campaign creation, data model, home screen with live data. Five issues, all shipped.

**Phase 2** — Engagement layer: streak indicators on campaign cards, activity history feed, stats dashboard, campaign archive. Four issues, all shipped.

**Phase 3** — Polish: onboarding flow, local push notifications, UI color palette refresh (softer brutalism, muted tones), shadow/border refinement, visual delight micro-interactions, progress sharing export. In flight.

That's somewhere between an MVP and a beta. Built autonomously. On a Saturday.

## What the $100 Is Actually Buying

The Claude Max subscription ($100/month) covers all five agents running continuously. There's no infrastructure cost — everything runs on my Mac via `launchd`. The only other costs are the GitHub repo and a Linear workspace (free tier works fine for this scale).

The real cost isn't money. It's prompt engineering time. Getting each agent to stay in its lane, communicate clearly in Linear, not make up Linear IDs, not break the lock file protocol — that took iteration. The prompts are living documents. When QA started running E2E tests that required an Android emulator, I had to update the prompt to handle emulator startup/teardown. When Designer started posting mockups that were too implementation-specific (bleeding into Programmer's territory), I had to add guardrails.

The agents are capable. The challenge is defining the interfaces between them precisely enough that they can actually collaborate.

## What I Didn't Expect

**The CTO memory file became the most useful artifact.** Each CTO run appends a log entry summarizing what was reviewed, what was approved, what phase the product is in. Reading those logs is the fastest way to understand where the product stands without opening Linear at all.

**Agents write better specs than I do.** When I write acceptance criteria, I write the minimum. When PM writes them, they're thorough — edge cases, error states, specific wording. When Designer writes design notes, they include animation timing values, scale transitions, color references back to the design system. It's better documentation than I would have written manually.

**The bottleneck isn't the agents, it's approval.** The workflow has an "Awaiting Design Approval" and "Awaiting Decision" status specifically because the agents can't make certain calls — judgment calls about product direction, design taste, scope. I still need to be in the loop for those. The company runs best when I batch my reviews: check in once, approve a cluster of things, let it run for hours.

**Sometimes an agent goes rogue in a funny way.** QA once tried to start an Android emulator, concluded it wasn't available, decided to use a physical device instead, then reported the physical device wasn't connected, then marked the tests as "inconclusive" and moved on. That was three iterations of prompt updates before it landed on "use `flutter test` for widget tests, don't launch an emulator unless the issue explicitly requires device testing."

## Would I Do It Again?

Yes, with one change: start with a simpler product scope. I gave the company "Campaign Tracker" as a concept, but a Flutter app with streaks, animations, and notifications is actually a moderately complex project. The agents handled it, but a simpler first product would have let me debug the orchestration layer faster.

The thing that makes this work isn't Claude being magic. It's the same thing that makes human teams work: clear roles, a shared task system, defined handoff points, and someone (me, apparently) who shows up periodically to make judgment calls and unblock things.

A $100 subscription. Five prompt files. A few hundred lines of shell script and Node.js. And now I have a company.

---

*The orchestration repo is at [levulinh/mtbox](https://github.com/levulinh/mtbox). The product repo is at [levulinh/mtbox-app](https://github.com/levulinh/mtbox-app).*
