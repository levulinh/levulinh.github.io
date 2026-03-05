---
layout: post
title: "China’s LLM Face-off: GLM-5 vs Qwen3.5 vs MiniMax-M2 — Who Wins in Code, Reasoning, and AI Agents?"
date: 2026-03-06
description: Conceptual diagram of a hybrid AI architecture using GLM-5 for reasoning and planning, Qwen 3.5 for multimodal UI and tools, and MiniMax M2.5 for high-speed code execution agents.
tags: AI LLM GLM-5 Qwen3.5 MiniMax-M2 MiniMax-M2.5 open-source coding agents benchmarks
giscus_comments: true
---

# China’s LLM Face-off: GLM-5 vs Qwen3.5 vs MiniMax-M2 — Who Wins in Code, Reasoning, and AI Agents?

If you’re doing serious work with open-weight models in 2026, your shortlist for “frontier-ish but still self-hostable” probably looks something like:

- **GLM-5** from Zhipu AI
- **Qwen 3.5** from Alibaba
- **MiniMax M2 / M2.5** from MiniMax

All three are positioned as *agent-ready*, *coding-competent*, and at least *partially enterprise-friendly*. Also, all three are being benchmarked to death on SWE-bench, Terminal-Bench, BrowseComp, and whatever new leaderboard someone spun up last week.

This post is a deeper, more numbers-heavy version of my earlier overview. Think of that one as the trailer; this is the director’s cut with the benchmark montages left in.

{%- include figure.liquid path="assets/img/blog/chinas-llm-face-off-glm-5-vs-qwen35-vs-minimax-m2-/04_illustration.png" alt="Three open-weight Chinese LLMs personified as robots in a coding arena" class="img-fluid rounded z-depth-1" -%}

You’re the target audience if you’re the sort of person who:

- Actually knows what **SWE-bench Verified**, **BrowseComp**, or **BFCL-V4** are,
- Runs your own vLLM server for fun,
- Has seriously considered burning an entire weekend wiring tools into yet-another agent framework.

Let’s cut through the marketing and look at where each model is *actually* strong, what the numbers say, and how to combine them without melting your GPU budget.

---

## 1. Quick Archetypes: Who’s Who?

Short character sheet, so we’re on the same page.

### GLM-5: Enterprise Nerd With a Low Hallucination Rate

- **Architecture:** ~744B-parameter MoE with ~40B active.
- **Design goal:** Long-horizon reasoning, coding, and **agentic sequences** with low hallucinations.
- **License:** MIT-style open weights.
- **Vibe:** "I read the docs, double-checked the spec, and I’m going to tell you when something smells off."

GLM-5 is the one you call when **being wrong is expensive**: enterprise, legal, critical systems, and any situation where “oops, I fabricated that API signature” is not acceptable.

### Qwen 3.5: Multimodal Polyglot Agent

- **Architecture:** Family from tiny (0.8B) to big MoE (397B-A17B reasoning variant).
- **Design goal:** Native *multimodal agent* — strong vision, multilingual, good tool use.
- **License:** Permissive Qwen license; broadly self-hostable and commercially usable.
- **Vibe:** "I’ll read your PDF, your UI screenshot, your Python, and your Chinese error log, and turn it into a workflow."

Qwen 3.5 is the most *generalist*: if you want a single family that does UI, docs, code, and agents reasonably well, this is the one.

### MiniMax M2 / M2.5: Born-for-Agents Code Monkey

- **Architecture:** MoE with ~230B total / ~10B active.
- **Design goal:** High-speed **coding** and **agentic workflows** with minimal serving cost.
- **License:** Open weights; via Hugging Face and NVIDIA NIM (NVIDIA Open Model License on NIM).
- **Vibe:** "Give me a repo, a test suite, and a backlog of bugs. I’ll grind."

M2.5 is explicitly optimized as a **coding+agent workhorse**: tuned with RL across hundreds of thousands of real environments, and marketed as SOTA on SWE-bench-class workloads at a fraction of closed-model prices.

---

## 2. Coding Benchmarks: Who Actually Fixes Your Tests?

Let’s start with the metric that tends to get funding approved: **does it fix real bugs in real repos?**

### 2.1 SWE-bench, Multi-SWE-Bench, and Friends

Below is a condensed comparison of the main public coding+agent benchmarks. Numbers are approximate and pulled from vendor reports plus third-party analyses.

> Caveat: different runs use different scaffolding agents, prompt templates, and tool stacks. Treat table values as *ballpark relative comparisons*, not lab-grade certs.

```text
Table 1 – Coding & Agent Benchmarks (as of early 2026, with tools/agents)
```

| Benchmark / Metric                  | GLM-5                          | Qwen (3.x / 3.5 coder variants)       | MiniMax M2.5                           | Notes |
|-------------------------------------|--------------------------------|----------------------------------------|-----------------------------------------|-------|
| **SWE-bench Verified** (% solved)  | ~77.8% with tools             | ~69–70% (Qwen3-Coder; 3.5 expected slightly higher) | **~80.2%** (vendor claim)             | M2.5 basically matches Claude Opus 4.6 / GPT-5.2 tier here. |
| **Multi-SWE-Bench** (% solved)     | n/a (not widely reported)     | n/a                                    | **~51.3%** (vendor, described as SOTA) | Strong evidence of robust multilingual code editing. |
| **SWE-rebench (Jan 2026, ~48 PRs)** | ~42.1%                        | ~40.0% (Qwen3-Coder-Next)             | ~39.6%                                  | Real GitHub PRs; all three in the same tier. |
| **Terminal-Bench 2.0** (agent)     | strong but less public detail | **~52.5** (Qwen 3.5)                  | very strong on harder variants          | Qwen 3.5 nears Gemini 3 Pro on terminal orchestration. |
| **Kilo.ai 3-task coding suite**    | high (slightly behind M2.5)   | not the focus of this report          | **88.5 / 100**, ~2× faster than GLM-5   | M2.5 finishes runs in ~21 min vs GLM-5’s ~44 min. |

A few takeaways:

- **MiniMax M2.5 is the current coding benchmark bully** among open weights.
  - Slightly higher **SWE-bench Verified** than GLM-5.
  - SOTA-ish **Multi-SWE-Bench** scores.
  - Kilo.ai finds it **both faster and more accurate** on practical tasks like bug hunting and large refactors.
- **GLM-5 is right behind** and sometimes ahead when the task becomes more reasoning-heavy.
  - On longer, more open-ended coding tasks, GLM-5 often produces more exhaustive, multi-file solutions.
- **Qwen 3.5’s story is more balanced**:
  - SWE-bench numbers lag GLM-5 and M2.5 a bit (based on Qwen3-Coder and early reports),
  - but it makes up ground in scenarios involving **terminal orchestration**, **function-calling**, and **multimodal inputs**.

If your workload resembles “SWE-bench + GitHub PR triage at scale”, M2.5 is the value pick. If it looks more like “design the system, then write the code”, GLM-5 becomes more attractive.

---

## 3. Reasoning & General Intelligence: Who You Ask to Design the System

Coding performance is one axis; **how well the model thinks** is another.

### 3.1 Artificial Analysis Intelligence Index

Artificial Analysis maintains a cross-bench **Intelligence Index** that mixes reasoning, math, coding, knowledge, and tool use.

```text
Table 2 – Reasoning & Tool-Use Benchmarks (selected)
```

| Benchmark / Metric                        | GLM-5                          | Qwen 3.5                              | MiniMax M2 / M2.5                 | Comments |
|-------------------------------------------|--------------------------------|----------------------------------------|-----------------------------------|----------|
| **Intelligence Index** (AA composite)     | **~50** (top open weights)    | ~45 (Qwen3.5-397B-A17B reasoning)     | strong, but exact score not public | GLM-5 is currently the strongest open-weight model here. |
| **Humanity’s Last Exam**                  | **~50.4**                      | smaller Qwen 3.5 often competitive     | good but less public data         | GLM-5 leads open weights on several reasoning-heavy suites. |
| **τ²-Bench** (telecom reasoning)         | **~89.7**                      | n/a                                   | n/a                               | GLM-5 shines at niche, structured reasoning. |
| **BrowseComp** (browsing + tools)        | **~75.9**                      | strong but not dominant                | **~76.3** (M2.5 with context mgmt) | GLM-5 and M2.5 are both top-tier here. |
| **BFCL-V4** (function calling)           | good                           | **~72.2** (Qwen3.5-122B-A10B)         | competitive                       | Qwen 3.5 beats GPT-5 mini (~55.5) on JSON/function calling. |
| **Terminal-Bench 2.0** (tool use/agents) | strong                         | **~52.5** (Qwen 3.5)                  | strong esp. on harder variants    | Qwen 3.5 nearly equals Gemini 3 Pro here. |

The headline:

- **GLM-5 is the reasoning champ** among open weights (for now).
  - Tops open weights on the **Intelligence Index**.
  - Leads on reasoning-heavy sets like **Humanity’s Last Exam** and **τ²-Bench**.
  - Extremely strong on **BrowseComp**, especially for long-horizon, tool-augmented browsing.
- **Qwen 3.5 is the function-calling & multimodal workhorse**.
  - That **72.2 BFCL-V4** on the 122B-A10B MoE is outstanding for robust JSON / tool calling.
  - Small 9B-ish models punch above their weight on GPQA and similar reasoning tests.
- **MiniMax M2/M2.5** show up very strong on agentic-style reasoning benchmarks.
  - Artificial Analysis notes **huge token volumes** during evaluation (~70M output tokens), implying aggressive chain-of-thought and tool usage.
  - M2.5’s **BrowseComp** numbers are right up there with GLM-5.

If you’re designing an LLM to write the architecture RFC, reason through edge cases, and then maybe hand off implementation, **GLM-5** is the safest open-weight bet.

---

## 4. Agentic Behavior & Tool Use: Who Plays Well With Others?

The current meta is “**model + tools + scaffold**”. So how do these three behave once you hand them a toolbox and a terminal?

### 4.1 Qwen 3.5: Native Agent Platform

Qwen 3.5 leans heavily into an **agent-first** design:

- **Terminal-Bench 2.0:** ~**52.5**, roughly on par with Gemini 3 Pro.
- **BFCL-V4:** ~**72.2** (122B-A10B), one of the best JSON/function-calling models.
- **Adaptive Tool Use:** built-in Search, Memory, and Code Interpreter tools in Qwen’s own stack.
- **Qwen-Agent:** an official agent framework showing multi-tool orchestration patterns.

In practice, this means less fiddling with whether the model will produce malformed JSON, or forget to call a tool even when it obviously should.

### 4.2 MiniMax M2.5: Born for Long Agent Loops

MiniMax literally brands M2/M2.5 as **"Mini model for Max coding & agentic workflows"**.

- RL-trained over **hundreds of thousands of real-world environments**.
- Strong scores on **Terminal-Bench Hard**, **SciCode**, and other agentic evaluations.
- Official marketing and third-party reports stress:
  - **Robust JSON & tool calling** in deeply nested loops.
  - Stable behavior even under long-running sessions.
- Benchmarks like **SWE-bench Verified**, **Multi-SWE-Bench**, and **BrowseComp** confirm that it doesn’t just call tools — it uses them effectively under load.

If you’re building a **continuous refactoring agent**, a **CI bot**, or anything that runs 24/7 inside your repos, M2.5 is engineered for that niche.

### 4.3 GLM-5: Agentic Intelligence With Preserved Thinking

GLM-5 takes a slightly different tack:

- Emphasizes **"agentic intelligence"** with *interleaved and preserved thinking modes*.
- API surfaces **`reasoning_content`** separately from user-facing `content`.
- Zhipu’s docs push a mantra like **"think in goals, not steps"** for agents.
- Leads open-source models on **Vending-Bench 2** (multi-turn ecommerce + browsing + purchases).

For framework authors, this is interesting: you can **capture and re-feed** the model’s reasoning traces across tool calls, effectively giving your agent a persistent inner monologue.

### 4.4 TL;DR on Agents

- **You want a batteries-included agent stack:** Qwen 3.5 (Qwen-Agent, strong function calling, Terminal-Bench).
- **You want industrial-grade coding agents:** MiniMax M2.5.
- **You want long-horizon planning with reusable thinking traces:** GLM-5.

---

## 5. Pricing, Context Windows, and Hardware Reality

Benchmarks are fun, but GPU invoices are real.

```text
Table 3 – Pricing, Context, and Throughput (approximate headline numbers)
```

| Model / API (flagship-ish)              | Input $ / 1M tokens | Output $ / 1M tokens | Context Window           | Throughput (tokens/s)      | Notes |
|-----------------------------------------|---------------------|----------------------|--------------------------|----------------------------|-------|
| **Qwen3.5-397B-A17B (reasoning)**      | ~**$0.60**          | ~**$3.60**           | up to **1M** (Qwen3.5-Plus API), 16k–128k for local open weights | ~86–89 t/s (AA tests)       | Expensive among open weights but still cheaper than many closed frontier models. |
| **GLM-5 (flagship)**                    | ~0.8× Qwen-Max input | ~0.6× Qwen-Max output | up to **200k**           | optimized on Ascend/Kunlunxin | Galaxy.ai shows ~20% cheaper input, ~40% cheaper output vs Qwen-Max. |
| **MiniMax M2**                          | ~**$0.255**         | ~**$1.00**           | ~**196k**                 | ~52.5 t/s                  | Pricing via PricePerToken; good speed for MoE. |
| **MiniMax M2.5 (std API)**             | similar ballpark, marketed as 10–20× cheaper than Claude/GPT/Gemini for SWE-bench-class workloads | | ~196k                     | ~54.3 t/s                  | High-speed variants exposed via API and NIM. |

A few reality checks:

- **Raw cost:** MiniMax M2/M2.5 are **noticeably cheaper** per token than Qwen 3.5 and GLM-5 in typical public pricing.
- **Context:**
  - Qwen 3.5 wins on sheer **theoretical max context** (up to 1M tokens in Qwen3.5-Plus),
  - GLM-5 and MiniMax both sit in the ~200k range, which is plenty for most agents.
- **Throughput:**
  - Qwen 3.5’s big reasoning model is fast given its size (~86–89 t/s),
  - MiniMax M2/M2.5 are very **speed-optimized** given they only activate ~10B parameters.

On-prem:

- **GLM-5** at full size is not a "throw it on a single 24GB card" kind of model. You’re looking at **multi-GPU** or serious clusters.
- **Qwen 3.5** and **MiniMax** offer **small and mid-size variants** that are realistically runnable on consumer GPUs:
  - Qwen 0.8B–9B, 27B, 35B-A3B via **Ollama/llama.cpp**.
  - MiniMax-M2 quantized on a single modern GPU.

For indie developers and small teams, that usually means:

- You use **smaller Qwen 3.5 + MiniMax M2** locally, and
- Maybe call GLM-5 or the big Qwen 397B remotely for heavy reasoning.

---

## 6. Ecosystem & Adoption: Who Actually Shows Up in Real Stacks?

Benchmarks are nice; **ecosystem gravity** often decides what you actually deploy.

### 6.1 GLM-5: Enterprise & China-Centric Gravity

- Strong traction in **Chinese enterprises**, especially where:
  - Data sovereignty and regulatory alignment matter.
  - There’s investment in **Huawei Ascend** or **Kunlunxin** accelerators.
- Available via Zhipu’s own API, **Together AI**, **SiliconFlow**, and others.
- Plays well with **vLLM** and popular inference stacks.
- MIT-style license is a big deal for organizations who hate vendor lock-in.

Less common in Western hobbyist / local-LLM circles, mostly because:

- The main GLM-5 model is huge,
- Smaller GLM-4.x variants exist but don’t carry the same “we’re at the top of the leaderboard” marketing.

### 6.2 Qwen 3.5: The Default Open-Weight VLM for Many Devs

- Hosted on **Hugging Face**, **ModelScope**, **Kaggle**.
- First-class support in **Ollama**, **vLLM**, **llama.cpp**, etc.
- **Qwen-Agent** provides reference patterns for tool use, browsing, and multi-tool orchestration.
- Widely integrated in LangChain, LlamaIndex, and generic “OpenAI-compatible” proxies.

For a lot of Western devs, Qwen 3.5 is now the de facto **local VLM + coder**: not perfect at anything, but very solid at almost everything.

### 6.3 MiniMax M2/M2.5: Agents in IDEs and Office Workflows

- Available via MiniMax’s API, aggregators, and **NVIDIA NIM** (`minimaxai/minimax-m2`).
- Open weights on **Hugging Face** with **vLLM** and **Ollama** support.
- Shows up in tutorials integrating with:
  - IDE agents (Cursor/Cline-style),
  - Office-style automation (Word/PPT/Excel generation),
  - Continuous repo maintenance bots.

Reviewers consistently describe M2/M2.5 as **"born for agents"**: high TPS, stable tool-calling JSON, and reasonable memory footprint despite strong coding performance.

---

## 7. Practical Model Selection: When to Pick Which

Let’s answer the actual question you care about: *what do I deploy for my use case?*

### 7.1 If You Care About Being Right Above All Else: GLM-5

Pick **GLM-5** when:

- You’re in a **regulated domain** (finance, healthcare, legal) where hallucinations are expensive.
- You need long-horizon **system design and reasoning** more than raw code-edit throughput.
- You want **MIT-licensed open weights** and control over deployment hardware.

Strengths:

- Best open-weight **reasoning** scores (Intelligence Index, Humanity’s Last Exam, τ²-Bench).
- Very strong **BrowseComp** and complex, tool-rich workflows.
- Enterprise-friendly license and narrative.

Trade-offs:

- Heavy hardware footprint at full scale.
- In pure SWE-bench-style coding throughput, MiniMax M2.5 will usually win.

### 7.2 If You’re Running a Coding Sweatshop (In a Nice Way): MiniMax M2.5

Pick **MiniMax M2.5** when:

- Your workload is **code-edit / run / fix** loops at scale:
  - Batch repo refactors
  - Continuous bug-fixing on live repos
  - CI-integrated agents running on every PR
- **Cost and speed** matter as much as top-1 solution quality.

Strengths:

- Top-tier **SWE-bench Verified** and **Multi-SWE-Bench** scores.
- Faster completion and better instruction adherence on practical coding tasks (per Kilo.ai).
- Aggressively tuned for **agent loops** and RL on real environments.

Trade-offs:

- Reasoning still good, but doesn’t quite match GLM-5’s depth on the hardest logic-heavy tasks.
- Not pitched as a frontier VLM; for visual reasoning you’ll likely pair it with a separate vision encoder or another model.

### 7.3 If You Want One Family to Rule Them All: Qwen 3.5

Pick **Qwen 3.5** when:

- You want a **single model family** for:
  - Chat & knowledge work,
  - Coding,
  - Vision (screenshots, PDFs, diagrams),
  - Agents with search/memory/code tools.
- You value **ease of deployment** across:
  - Local machines (0.8B–9B/27B/35B),
  - Cloud APIs, and
  - OSS tools like Ollama, vLLM, etc.

Strengths:

- Great **multimodal** capabilities.
- Strong **Terminal-Bench 2.0** and **BFCL-V4** scores.
- Small and mid-sized models that punch above their weight.

Trade-offs:

- On pure SWE-bench Verified, looks a bit weaker than GLM-5 and M2.5 (using current Qwen coder data).
- Flagship 397B reasoning model is not cheap; still cheaper than closed frontier, but not budget-tier.

---

## 8. Hybrid Architectures That Don’t Hate You

In practice, many serious teams end up with **more than one** of these models. The patterns that keep showing up in evaluations and practitioner write-ups are hybrid.

Here’s the mental model I like:

{%- include figure.liquid path="assets/img/blog/chinas-llm-face-off-glm-5-vs-qwen35-vs-minimax-m2-/05_illustration.png" alt="Diagram of a hybrid architecture stacking GLM-5, Qwen 3.5, and MiniMax M2.5" class="img-fluid rounded z-depth-1" -%}

### Pattern A: Qwen for Frontend, GLM-5 for Planning, MiniMax for Execution

1. **Multimodal UI / Intake — Qwen 3.5**
   - User uploads screenshots, PDFs, error logs, or code snippets.
   - Qwen 3.5:
     - Parses visuals,
     - Extracts structured tasks,
     - Handles multilingual input if needed.

2. **Deep Planning & Design — GLM-5**
   - Qwen 3.5 hands a structured problem description to GLM-5.
   - GLM-5:
     - Designs the approach (e.g., refactor plan, migration steps),
     - Identifies edge cases, risks, and test strategies,
     - Produces a **goal-oriented plan**.

3. **High-Throughput Execution — MiniMax M2.5**
   - M2.5 gets:
     - The high-level plan from GLM-5,
     - The actual repo context and tests.
   - It then runs the **edit → run → fix loop** using tools (terminal, git, test runners), optimizing for speed and cost.

This gives you:

- Qwen’s strengths at the **UI and multimodal boundary**,
- GLM-5’s **trustworthy reasoning** where it matters most,
- M2.5’s **brutal efficiency** on code changes.

### Pattern B: Local-First Dev Setup (Indie / Small Team)

You probably don’t have a 744B MoE under your desk. A more realistic stack:

- **Laptop / workstation:**
  - Run **Qwen3.5-9B** (or 27B) locally via Ollama for:
    - Chat, code suggestions, quick multimodal inspections.
  - Run a **quantized MiniMax-M2** via vLLM for:
    - Local coding agents handling your main repos.
- **Cloud fallback:**
  - Use GLM-5 or Qwen3.5-397B only for the big, gnarly planning queries.

Your router might look something like this:

```python
# Pseudo-code: simple triage router

def route_request(task):
    if task.type == "multimodal" or task.language != "en":
        return "qwen-3.5-9b-local"  # screenshots, PDFs, multilingual bugs
    
    if task.type == "deep_planning" and task.impact == "high":
        return "glm-5-cloud"        # system design, critical migrations

    if task.type in {"bug_fix", "refactor", "swe-bench-like"}:
        return "minimax-m2.5"       # fast edit-run-fix loops

    return "qwen-3.5-9b-local"      # default assistant
```

Is this oversimplified? Absolutely. Does it beat “just always call GPT-Whatever” in cost/performance for many teams? Increasingly, yes.

### Pattern C: GLM-5-First Enterprise Stack

For more conservative orgs that want **one primary brain**:

- Use **GLM-5** as the default model for:
  - Internal chat,
  - Knowledge retrieval glue (RAG),
  - System design and doc review,
  - Compliance-sensitive workloads.
- Plug in **MiniMax M2.5** behind a "coding agent" microservice:
  - GLM-5 writes out execution plans and code change specs.
  - M2.5 executes those plans at scale, because nobody wants to pay GLM-5 rates for updating 300 microservices.
- Optionally, use a **smaller Qwen 3.5** model for edge multimodal cases (UI screenshots, raw PDFs) if GLM’s multimodal variants aren’t enough.

---

## 9. Caveats, Missing Data, and What to Watch

A few reality checks before we pretend this is all settled science:

- **Benchmark coverage is patchy.**
  - We have detailed **Multi-SWE-Bench** numbers for MiniMax, but not for GLM-5 and Qwen 3.5 yet.
  - Some of the more exotic reasoning benchmarks (like niche telecom datasets) only have GLM-5 numbers.
- **Vendor vs third-party numbers.**
  - Some results (e.g., M2.5’s 80.2% SWE-bench Verified, 51.3% Multi-SWE-Bench, 76.3 BrowseComp) are **vendor-reported** but align reasonably with independent checks.
- **Agent scaffolding matters a lot.**
  - SWE-bench with a well-tuned agent can be 20+ points higher than with a naive wrapper.
  - When you see small deltas (e.g., 39.6% vs 42.1% on SWE-rebench), they might be within the variance introduced by your own prompt orchestration.

So treat the numbers as **priors**, not ground truth. You’ll still want to run your own evals on your stack.

---

## 10. So… Who Wins?

In the strict “leaderboard or death” sense:

- **Coding + cost efficiency:** MiniMax M2.5
- **Reasoning + low hallucination + enterprise story:** GLM-5
- **Multimodal + tooling + ecosystem:** Qwen 3.5

But that’s not actually how serious teams choose models. The more actionable summary is:

- If you’re an **enterprise** building **critical long-horizon agents**:
  - Start with **GLM-5**.
  - Add **MiniMax M2.5** for high-volume coding.
  - Sprinkle in **Qwen 3.5** where you need rich multimodal intake.
- If you’re a **startup or indie dev**:
  - Run **Qwen 3.5 (small/mid)** and **MiniMax M2** locally.
  - Call GLM-5 or big Qwen only when you truly need frontier-level reasoning.
- If you’re building **agent frameworks or dev tools**:
  - Design your system to **treat models as pluggable roles**, not a single monolith.
  - GLM-5 for planning, Qwen 3.5 for multimodal I/O, M2.5 for execution is a very reasonable default trio.

The good news: we’re finally in a world where **open-weight models from a single region (China) are competitive across the entire stack** — coding, reasoning, and agentic workflows — and you can mix and match them with real architectural intent instead of just copying whatever closed model the last blog post used.

The bad news: you now have three more knobs to tune.

Choose wisely — and more importantly, measure ruthlessly.
