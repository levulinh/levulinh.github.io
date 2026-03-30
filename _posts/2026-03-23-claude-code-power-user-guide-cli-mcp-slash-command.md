---
layout: post
title: "Claude Code Power User Guide: CLI, MCP, Slash Commands, Hooks, and Git Worktrees"
date: 2026-03-23
description: "A practical deep-dive into Claude Code's most powerful features — CLAUDE.md, custom slash commands, MCP servers, hooks, and Git Worktrees for parallel sessions — with real examples and workflow tips for developers."
tags:
  - "Claude Code"
  - "AI tools"
  - "developer productivity"
  - "MCP"
  - "CLI"
giscus_comments: true
---

I've been using Claude Code long enough to have strong opinions about it. It's not a magic code generator — it's a terminal-native agentic CLI that slots into how you actually work, if you invest a few hours configuring it properly. This post covers the features that actually move the needle: CLAUDE.md, slash commands, hooks, MCP servers, and Git Worktrees for parallel sessions.

{%- include figure.html path="assets/img/blog/claude-code-power-user-guide-cli-mcp-slash-command/03_illustration.jpg" alt="Claude Code CLI terminal interface with MCP connections and glowing developer workspace" class="img-fluid rounded z-depth-1" -%}

## CLAUDE.md: The File That Changes Everything

If there's one thing to set up before anything else, it's `CLAUDE.md`. This file is how Claude persists knowledge across sessions — your coding standards, architecture decisions, forbidden commands, and whatever tribal knowledge would otherwise live in a README nobody reads.

A 3-layer hierarchy keeps things manageable:

- **Global** (`~/.claude/CLAUDE.md`) — applies to every project. Good for personal preferences, editor setup, preferred testing patterns.
- **Project** (`.claude/CLAUDE.md` or `CLAUDE.md` at repo root) — team conventions, build commands, architecture notes.
- **Task** (inline during session) — scratch notes or context specific to what you're doing right now.

Here's a minimal but useful project-level `CLAUDE.md`:

```markdown
# Project: payments-service

## Build & test
- Run tests: `pnpm test`
- Lint: `pnpm lint` (must pass before any commit)
- Build: `pnpm build`

## Architecture
- All DB access goes through `src/db/` — never raw queries in handlers
- Use `Result<T, AppError>` pattern; no thrown exceptions in business logic
- Feature flags live in `src/config/flags.ts`

## Off-limits
- Never run `DROP TABLE` or any destructive migration without explicit confirmation
- Never commit `.env` files or secrets
```

One nuance worth knowing: Claude Code's system prompt and tool definitions already burn ~19K tokens before your conversation starts — roughly 10% of the 200K context window. Keep `CLAUDE.md` tight. It's not a wiki; it's a briefing document.

## Slash Commands and Hooks: Automating Your Workflow

{%- include figure.html path="assets/img/blog/claude-code-power-user-guide-cli-mcp-slash-command/04_illustration.jpg" alt="Claude Code ecosystem diagram showing CLAUDE.md, Git Worktrees, MCP Servers, Slash Commands, and Hooks connected in a developer workflow" class="img-fluid rounded z-depth-1" -%}

Slash commands turn multi-step workflows into single-line invocations. Drop a Markdown file into `.claude/commands/` and it becomes a callable command.

```
.claude/
  commands/
    review-pr.md
    deploy-staging.md
    db-migrate.md
```

A `review-pr.md` might look like this:

```markdown
---
description: Review the current PR for security issues, logic bugs, and test coverage
allowed-tools: [Bash, Read, Grep]
---

Review the diff from `git diff main...HEAD`.

Check for:
1. Security issues (SQL injection, unvalidated input, leaked secrets)
2. Logic bugs and edge cases
3. Missing test coverage for changed paths
4. Anything that violates the conventions in CLAUDE.md

Output a structured report with severity labels: CRITICAL / WARN / INFO.
```

Then you just type `/review-pr` and Claude picks it up. Global commands go in `~/.claude/commands/` if you want them across all projects.

**Hooks** are the other half of this equation. They let you run shell scripts before or after Claude Code actions — think of them as lifecycle middleware. The hook events are `PreToolUse`, `PostToolUse`, `PermissionRequest`, and `ConfigChange`.

Configure interactively with `/hooks`, or write the JSON directly in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "pnpm lint --fix $CLAUDE_FILE_PATHS"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo $CLAUDE_TOOL_INPUT | grep -Eq 'DROP TABLE|rm -rf /' && echo 'BLOCKED' && exit 1 || exit 0"
          }
        ]
      }
    ]
  }
}
```

This setup auto-lints after every file write and blocks a handful of destructive bash patterns before they run. You can even define `type: agent` hooks that spin up a full sub-agent with Read/Grep/Glob access to perform more sophisticated checks — effectively a code quality gate that runs automatically without you asking.

## MCP Servers Worth Installing Today

MCP turns Claude into a proper tool-orchestrating agent. The servers I actually use and trust:

| Server | What it does |
|---|---|
| **GitHub MCP** | PR creation, issue management, repo browsing — right from chat |
| **Supabase MCP** | Live DB operations, schema inspection, query execution |
| **Firecrawl MCP** | Scrapes URLs into clean LLM-ready Markdown |
| **E2B MCP** | Runs code in a secure cloud sandbox — great for untrusted codegen |
| **Context7 MCP** | Fetches up-to-date library docs instead of Claude hallucinating v1.2 APIs |
| **Kubernetes MCP** | Cluster CRUD, pod inspection, log streaming |

Connect a server in `.claude/settings.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

Check what's connected at any time with `/mcp`. A word of caution: load only the servers you need for the current task. Every connected MCP server contributes to your context budget, and an agent that can push to GitHub and hit your prod database simultaneously is carrying real blast radius.

For discovery, [mcpmarket.com](https://mcpmarket.com) is a decent directory. The [ykdojo/claude-code-tips](https://github.com/ykdojo/claude-code-tips) repo (6K+ stars) also has a curated list.

## Git Worktrees for Parallel Sessions

This one genuinely changed how I work. Git Worktrees let you have multiple checkouts of the same repo, each on its own branch, sharing the same `.git` directory. No stashing, no context switching, no file conflicts.

Combine that with Claude Code and you can run 10–15 sessions simultaneously — one implementing a feature, another writing tests, another doing a refactor — all without stepping on each other.

```bash
# Create a worktree for a new feature branch
git worktree add ../payments-service-auth feature/oauth-refresh
cd ../payments-service-auth
claude  # Fresh Claude Code session, isolated branch
```

Or just ask Claude directly: *"Create a worktree for a new branch called feature/rate-limiting and start working on it."* It handles the `git worktree add` incantation for you.

The typical parallel workflow:

1. Main worktree — active development
2. Second worktree — Claude running TDD on the new endpoint
3. Third worktree — Claude drafting a migration and waiting for your review

Each session has its own context, its own terminal, and zero interference. Clean up when done:

```bash
git worktree remove ../payments-service-auth
```

## Context Management and the Workflow That Actually Works

Context is finite and gets messy fast. A few habits that help:

- **`/compact`** — ask Claude to condense the conversation into a summary before you hit the limit. Do this proactively, not when you're desperate.
- **`/clear`** — start fresh when the current task is genuinely done and the next one is unrelated. Carrying stale context into a new task is just noise.
- **Plan Mode first** — hit `Shift+Tab` (or `--permission-mode plan`) to put Claude in explore-only mode. Ask it to think through a few approaches, pick the simplest one, then generate a spec with numbered milestones before writing a single line of code. Mid-task corrections are expensive; a bad plan at the start is worse.

The workflow I've landed on for non-trivial tasks:

1. `/clear` to start fresh
2. Plan Mode → explore approaches → agree on one
3. Ask Claude to generate a spec and convert it to a numbered to-do list
4. Execute step by step, checking off tasks — Claude has a concrete target to verify against
5. Write tests first if it's a greenfield function; let Claude run the suite and iterate

For the brave: `--dangerously-skip-permissions` skips the constant permission prompts in trusted local sessions. Useful when you're iterating fast on a throwaway branch. Absolutely do not use it pointed at production anything.

Claude Code also auto-saves key learnings across sessions — things like your build commands and test runner invocations get remembered without manual effort. The memory is lightweight but it means session three doesn't feel like session one.

The tooling is genuinely good now. The gap between a developer who's configured it properly and one who's just running `claude` cold is massive — probably an order of magnitude in task throughput. The few hours it takes to set up `CLAUDE.md`, a handful of slash commands, and two or three MCP servers is the best ROI you'll get from any dev tool this year.