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

{% include figure.html path="assets/img/blog/mtbox-virtual-company/dashboard.png" alt="The MTBox agent dashboard — five agents running autonomously on a polling schedule" class="img-fluid rounded z-depth-1" %}

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

{% include figure.html path="assets/img/blog/mtbox-virtual-company/linear-board.png" alt="Linear issue board showing Campaign Tracker tasks across all workflow stages" class="img-fluid rounded z-depth-1" %}

Each agent identifies itself in comments with a prefix: `📋 [PM]`, `🎨 [Designer]`, `💻 [Programmer]`, `🔍 [QA]`, `🏢 [CTO]`. This makes the activity feed readable as a conversation — which it effectively is.

## Watching a Feature Ship

Here's what it looks like in practice. I gave the CTO a directive: _"Add some touches so the app doesn't feel too boring."_ The CTO translated that into a Phase 3 roadmap item — "Visual delight polish: micro-interactions, empty states, subtle transitions" — and dropped it in the backlog.

PM picked it up, wrote acceptance criteria, and moved it to "In Design."

Designer opened the issue, built a mockup in Flutter, and posted it with detailed panel-by-panel specs:

{% include figure.html path="assets/img/blog/mtbox-virtual-company/linear-issue-comments.png" alt="Designer Bot posting a Flutter mockup with design notes for the Visual Delight Polish issue" class="img-fluid rounded z-depth-1" %}

Then CTO reviewed the design and either approved or pushed back. In this case: approved, with notes on which acceptance criteria were satisfied. Programmer picked it up, implemented it, and opened a PR. QA ran 20 unit and widget tests — all passed.

{% include figure.html path="assets/img/blog/mtbox-virtual-company/linear-issue-comments2.png" alt="Programmer Bot reporting implementation complete, QA Bot reporting 20/20 tests passing" class="img-fluid rounded z-depth-1" %}

The whole thing — from directive to green tests — happened while I wasn't watching. That's the part that still surprises me every time I check in.

## The Evolution: Six Rounds of "This Isn't Quite Right"

The first version, built at midnight, was embarrassingly simple: four shell scripts, four prompt files, a `launchd` plist to wake each agent every 15 minutes. That was it. No dashboard. No CTO. No webhook. Just four processes polling Linear on a timer and posting comments.

It worked, sort of. But within a few hours I was watching logs with `tail -f` on four terminal windows simultaneously, squinting to figure out which agent was doing what. So I built a dashboard — a Node.js server with a brutalist-looking HTML frontend: five agent cards, IDLE/BUSY status badges, a countdown timer to each agent's next run, and a live log stream panel. Now instead of four terminals, I had one ugly webpage that told me everything at a glance.

That helped. But the next problem surfaced quickly: I was still the PM's PM. Every time I had a new feature idea, I had to manually write a Linear issue for it. The agents could execute work; they couldn't figure out what work needed doing. So I added a CTO agent — a fifth agent sitting between me and the PM. I write high-level directives ("make the app feel less boring"). CTO turns them into a phased roadmap and seeds tasks in the backlog. PM picks up from there. The chain of interpretation that happens in a real company — CEO vision → CTO strategy → PM execution — now happened without me in the middle of it.

The next friction was speed. With two-hour polling intervals, there could be a three-hour gap between the Designer finishing a mockup and the Programmer noticing it. That's fine overnight, miserable during active development. I added a Cloudflare quick tunnel and registered a Linear webhook: status changes now fire the right agent immediately, with polling as a silent backup.

At some point I looked at my token usage and winced. PM and QA were running on Sonnet — the same model as the Programmer who was writing real Dart code. That's like putting a senior engineer on Jira ticket grooming duty. Switching PM and QA to Haiku cut costs noticeably without any visible drop in quality. I also stretched the polling intervals from 15 minutes to 2 hours, since the agents were mostly running idle checks anyway.

The last piece was the PAUSE button. Sometimes I want to edit a prompt or push changes to the orchestration repo without agents firing mid-edit. A pause toggle with an auto-resume timer turned the system from something I watched nervously into something I actually felt in control of.

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

## What This Is Not

I want to be honest about the limits here, because it's easy to over-read what's actually happening.

These agents are genuinely useful, but they are also genuinely naive. They follow prompts, not principles. They don't understand why something matters — they understand what they've been instructed to do. When the task matches the prompt, they're great. When reality drifts from what the prompt anticipated, they either hallucinate their way forward or get stuck in a loop and time out.

More concretely: nothing in this system is handling the problems that actually make software hard. Security isn't considered. The Programmer writes code that compiles and passes widget tests, but nobody has audited the data persistence layer, thought carefully about authentication, or reviewed for injection risks. Scaling isn't on anyone's radar — the QA agent runs `flutter test`, not load tests. There's no incident response. No observability beyond "did the build pass." The agents write code the way a new grad might: functionally correct, structurally reasonable, not battle-tested.

For a personal side project, that's fine. For anything with real users or real data, you'd need a human engineer reviewing every PR with genuine critical thinking — not just a QA agent running the test suite. The agents can move fast precisely because they're not asking the hard questions. A human has to ask those.

This also doesn't scale in the org-chart sense. Five agents coordinating through Linear comments works because the product is small and the team is small. Add more agents, more products, more cross-cutting concerns, and the coordination overhead would start to eat the output. There's no real escalation path when something goes genuinely wrong. When QA fails a PR, the Programmer fixes it — but nobody asks whether the acceptance criteria themselves were correct. The system is good at executing decisions, not at questioning them.

I think of it as a force multiplier for a solo developer, not a replacement for a team. It handles the mechanical parts of software development — the tickets, the boilerplate, the test runs, the PR descriptions — so I can focus on the parts that actually require judgment.

## Would I Do It Again?

Yes, with one change: start with a simpler product scope. I gave the company "Campaign Tracker" as a concept, but a Flutter app with streaks, animations, and notifications is actually a moderately complex project. The agents handled it, but a simpler first product would have let me debug the orchestration layer faster.

The thing that makes this work isn't Claude being magic. It's the same thing that makes human teams work: clear roles, a shared task system, defined handoff points, and someone (me, apparently) who shows up periodically to make judgment calls and unblock things.

A $100 subscription. Five prompt files. A few hundred lines of shell script and Node.js. And now I have a company — of sorts.

---

_The orchestration repo is at [levulinh/mtbox](https://github.com/levulinh/mtbox). The product repo is at [levulinh/mtbox-app](https://github.com/levulinh/mtbox-app)._
