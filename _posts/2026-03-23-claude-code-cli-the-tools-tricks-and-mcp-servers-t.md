---
layout: post
title: "Claude Code CLI: The Tools, Tricks, and MCP Servers That Actually Matter"
date: 2026-03-23
description: "A practical guide to Claude Code's most useful features — CLAUDE.md memory, Plan Mode, MCP servers, custom slash commands, hooks, and Git worktrees for parallel sessions — with real commands and config examples."
tags:
  - "Claude Code"
  - "developer productivity"
  - "MCP"
  - "CLI"
  - "AI tools"
giscus_comments: true
---

I've spent enough time with Claude Code to separate the features worth caring about from the ones that look good in demos. This is the latter filtered out. Here's what actually changes how you work.

{%- include figure.html path="assets/img/blog/claude-code-cli-the-tools-tricks-and-mcp-servers-t/03_illustration.jpg" alt="Claude Code CLI terminal workspace connected to GitHub, Docker, Supabase and Vercel via glowing MCP nodes" class="img-fluid rounded z-depth-1" -%}

## Getting Started with Claude Code CLI

Install it globally via npm:

```bash
npm install -g @anthropic-ai/claude-code
```

Then from any project directory, just run:

```bash
claude
```

That drops you into an interactive REPL. Claude can read your files, run bash commands, edit code, and call external tools — all without leaving the terminal. It's not a chat interface with a code block you copy-paste from; it's a genuine agent loop.

For quick one-shot queries, skip the REPL entirely:

```bash
# One-liner query with plain text output
claude -p "What does the auth middleware do?" --output-format text

# Pipe a file in and get structured output back
cat src/api/users.ts | claude -p "Find all unvalidated inputs" --output-format json
```

That `--output-format json` flag is underrated. It makes Claude a proper Unix-style utility — you can pipe its output into `jq`, shell scripts, CI steps, whatever. I use it in a pre-commit hook that scans staged files for secrets and hardcoded credentials.

For riskier tasks — long refactors, file deletions, anything irreversible — run Claude inside a container:

```bash
docker run -it --rm -v $(pwd):/app -w /app node:20 npx @anthropic-ai/claude-code
```

Isolation is free. Use it.

## CLAUDE.md: Your Project's Persistent Memory

Every Claude Code session starts cold. `CLAUDE.md` is how you fix that — it's a Markdown file that Claude reads at the start of every session, giving it your conventions, constraints, and project-specific knowledge without you having to repeat yourself.

Three levels:

- **Global** (`~/.claude/CLAUDE.md`) — applies everywhere: your editor setup, preferred testing patterns, general style
- **Project** (`.claude/CLAUDE.md` or repo root `CLAUDE.md`) — team conventions, build commands, architecture rules
- **Task-level** — inline context you add mid-session for temporary guidance

A minimal but genuinely useful project `CLAUDE.md`:

```markdown
# payments-service

## Commands
- Test: `pnpm test`
- Lint (must pass before commit): `pnpm lint`
- Dev server: `pnpm dev`

## Architecture
- All database access through `src/db/` — no raw queries in route handlers
- Use `Result<T, AppError>` everywhere; never throw in business logic
- Feature flags live in `src/config/flags.ts`

## Hard rules
- Never run destructive migrations without explicit confirmation
- Never touch `.env` files
- Always write tests before asking me to merge
```

One thing to keep in mind: Claude Code's system prompt and tool definitions consume around 19K tokens before your conversation even starts — roughly 10% of the 200K context window. Keep `CLAUDE.md` concise. It's a briefing document, not a wiki.

Plan Mode is the other thing worth setting up as a habit. Hit `Shift+Tab` (or pass `--permission-mode plan`) and Claude switches to read-only exploration mode — no writes, no commands, just analysis. Use it to understand a codebase before touching it, or to agree on an approach before Claude starts generating code you'll have to undo.

```bash
claude --permission-mode plan
```

## MCP Servers: Plugins That Actually Extend Claude

MCP (Model Context Protocol) connects Claude to external tools — databases, APIs, browsers, cloud services — through a standardised interface. Think of them as plugins with a real spec.

Add a server via CLI or directly in `.claude/settings.json`:

```bash
claude mcp add context7 -- npx -y @upstash/context7-mcp
claude mcp add github -- npx -y @modelcontextprotocol/server-github
```

Or configure directly:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    },
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase"],
      "env": { "SUPABASE_URL": "${SUPABASE_URL}", "SUPABASE_KEY": "${SUPABASE_KEY}" }
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

The servers I'd actually recommend installing:

| Server | What it does |
|---|---|
| **Context7** | Injects live, version-specific library docs into context — stops Claude hallucinating deprecated APIs |
| **GitHub MCP** | PR creation, issue management, repo browsing from chat |
| **Supabase MCP** | Live DB operations, schema inspection, query execution |
| **Playwright MCP** | Browser automation and E2E test execution |
| **Firecrawl MCP** | Scrapes URLs into clean Markdown — useful for pulling in docs or changelogs |
| **E2B MCP** | Runs generated code in a secure cloud sandbox before it touches your machine |
| **Sequential Thinking MCP** | Forces structured multi-step reasoning — noticeably better on complex planning tasks |

Check what's connected at any point with `/mcp`. One caution: every active MCP server eats into your context budget and expands your blast radius. An agent that can simultaneously push to GitHub and query your prod database is carrying real risk. Load only what the current task needs.

## Custom Slash Commands and Hooks

Slash commands turn repetitive multi-step prompts into reusable one-liners. Create a Markdown file in `.claude/commands/` and it becomes callable:

```
.claude/
  commands/
    commit.md
    review-pr.md
    db-migrate.md
```

A `commit.md` that generates conventional commits:

```markdown
---
description: Generate a conventional commit message and commit staged changes
allowed-tools: [Bash, Read]
---

Look at `git diff --staged` and write a conventional commit message following this format:
`type(scope): description` where type is one of: feat, fix, docs, refactor, test, chore.

Keep the subject line under 72 characters. Add a body if the change is non-obvious.
Then run `git commit -m "<message>"` with the generated message.
```

Now `/commit` does the whole thing. Global commands go in `~/.claude/commands/` if you want them across all projects.

**Hooks** are lifecycle scripts — shell commands that fire at specific Claude Code events. Configure them in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "pnpm lint --fix $CLAUDE_FILE_PATHS" }]
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

This auto-lints every file Claude touches and hard-blocks a handful of dangerous bash patterns. You can also set up a notification hook so your terminal pings you when Claude needs input — useful when you've handed off a long task and stepped away:

```json
{
  "hooks": {
    "PermissionRequest": [
      {
        "hooks": [{ "type": "command", "command": "terminal-notifier -message 'Claude needs your input'" }]
      }
    ]
  }
}
```

## Power Tips: Worktrees, Context, and the Workflow That Scales

{%- include figure.html path="assets/img/blog/claude-code-cli-the-tools-tricks-and-mcp-servers-t/04_illustration.jpg" alt="Parallel git worktrees running independent Claude Code sessions on separate branches" class="img-fluid rounded z-depth-1" -%}

Git worktrees are the most underused feature in a parallel Claude workflow. A worktree is a second (or third, or tenth) checkout of the same repo on a different branch, sharing one `.git` directory. No stashing, no conflicts, no switching.

```bash
# Spin up an isolated worktree for a new feature
git worktree add ../my-project-auth feature/oauth-refresh
cd ../my-project-auth
claude  # fresh session, isolated branch
```

Run three sessions simultaneously — one on feature work, one writing tests, one doing a refactor — and they never interfere. Clean up when done:

```bash
git worktree remove ../my-project-auth
```

Context management is the other thing that separates productive sessions from ones that slowly go sideways:

- **`/compact`** — asks Claude to summarise the conversation into a condensed form. Do this proactively every 30–40 messages, not when you're already hitting the limit.
- **`/clear`** — full reset. When one task is done and the next is genuinely unrelated, start clean. Stale context makes Claude less useful, not more.
- **`/stats`** — shows current token usage. Worth checking on long sessions so you're not surprised.

The workflow I've settled on for non-trivial tasks:

1. `git commit` everything — so you can instantly revert
2. `/clear` for a fresh session
3. Plan Mode first: explore the codebase, agree on an approach
4. Generate a spec, break it into a numbered to-do list
5. Execute incrementally; let Claude verify each step against the list

For fully automated pipelines, Claude Code's SDK can be triggered from GitHub Actions, Slack bots, or monitoring alerts — receive an alert, spin up a Claude agent, get back a tested PR with no human in the loop. That's the end game, and it's not as far off as it sounds.

The gap between a configured Claude Code setup and a bare `claude` invocation is genuinely large. A few hours on `CLAUDE.md`, two or three MCP servers, and a handful of slash commands will compound across every session after. That's about the best ROI available from any dev tool right now.