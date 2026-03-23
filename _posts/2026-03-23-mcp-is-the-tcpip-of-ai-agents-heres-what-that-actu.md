---
layout: post
title: "MCP Is the TCP/IP of AI Agents — Here's What That Actually Means"
date: 2026-03-23
description: "MCP (Model Context Protocol) is the open standard quietly becoming the universal connectivity layer for AI agents. Here's the engineering breakdown of why the TCP/IP analogy holds, what the architecture looks like, and why you should care right now."
tags:
  - "AI agents"
  - "MCP"
  - "protocols"
  - "LLM"
  - "engineering"
giscus_comments: true
---

If you've spent any time around AI tooling in the past six months, you've heard someone say "MCP is the TCP/IP of AI agents." It sounds like marketing fluff until you sit with it for five minutes — then it clicks, and you realize it's actually a pretty sharp analogy. Let me break down what MCP is, why the comparison holds, and what you should do about it.

{%- include figure.html path="assets/img/blog/mcp-is-the-tcpip-of-ai-agents-heres-what-that-actu/01_illustration.jpg" alt="NxM integration problem vs. MCP solution — before and after diagram" class="img-fluid rounded z-depth-1" -%}

## 1. The Problem It Solves: AI's Own "NxM Hell"

Cast your mind back to networking before TCP/IP. Every vendor had their own proprietary protocol. Connecting two machines from different manufacturers was a custom engineering project every single time. Then TCP/IP showed up, gave every device a common language, and suddenly the internet became possible.

AI tooling in 2024 had the same problem. If you wanted Claude to query your database, you wrote a custom integration. If you then wanted GPT-4 to do the same thing, you wrote *another* custom integration. If you added a new tool — say, a GitHub connector — you wrote N more integrations, one per model. This is the **NxM problem**: N models × M tools = N×M one-off integrations. It's the kind of thing that sounds manageable until your third one, at which point you start questioning your career choices.

MCP, released by Anthropic in late 2024, collapses that to **N+M**. Each AI model implements the MCP client protocol once. Each tool (database, API, filesystem, whatever) implements the MCP server once. Everyone talks the same language. The combinatorial explosion goes away.

Same move TCP/IP made. Same payoff.

## 2. How It Actually Works: The Architecture

MCP is built on JSON-RPC 2.0 and follows a three-part architecture: **Host → Client → Server**.

- **Host**: The application shell — Claude Desktop, VS Code, your custom chat app. It manages one or more MCP clients and handles user-level permissions.
- **Client**: Maintains a stateful 1:1 session with a specific MCP server. The session is persistent, not fire-and-forget REST calls.
- **Server**: Exposes capabilities via three primitives:
  - **Tools** — callable functions the agent can invoke (think: `run_sql_query`, `create_github_issue`)
  - **Resources** — data or files the agent can read (think: filesystem paths, database schemas)
  - **Prompts** — reusable prompt templates the server pre-packages

Here's what a minimal MCP server looks like in Python:

```python
from mcp.server import MCPServer
from mcp.types import Tool, TextContent

server = MCPServer("my-db-server")

@server.tool()
async def run_query(sql: str) -> list[TextContent]:
    """Execute a read-only SQL query and return results."""
    results = await db.execute(sql)
    return [TextContent(type="text", text=str(results))]

if __name__ == "__main__":
    server.run()  # defaults to stdio transport
```

And the client side, connecting to it:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async with stdio_client(StdioServerParameters(command="python", args=["my_db_server.py"])) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()          # capability handshake
        tools = await session.list_tools()  # discover available tools
        result = await session.call_tool("run_query", {"sql": "SELECT * FROM users LIMIT 10"})
```

That initialization handshake — where client and server explicitly declare what capabilities they support — is MCP's equivalent of TCP's three-way handshake. It's what makes forward/backward compatibility possible: a 2025 client talks fine to a 2026 server because they negotiated what they both understand upfront.

For transport, MCP gives you two options: **stdio** for local processes (fast, simple, great for dev tooling) and **HTTP with Server-Sent Events** for remote/cloud deployments (scales, firewalls, all that fun enterprise stuff).

## 3. The Bigger Stack: Where MCP Fits

Here's where it gets really interesting. MCP is just one layer of an emerging agent protocol stack.

{%- include figure.html path="assets/img/blog/mcp-is-the-tcpip-of-ai-agents-heres-what-that-actu/02_illustration.png" alt="Agent Protocol Stack — layered like a TCP/IP or OSI stack" class="img-fluid rounded z-depth-1" -%}

Think of it as:

1. **Transport layer** — stdio / HTTP+SSE (how bytes move)
2. **MCP layer** — model-to-tool access (how agents talk to data and services)
3. **A2A layer** — Google's Agent-to-Agent protocol (how agents delegate tasks to *other agents*)
4. **A2UI layer** — how agents render output to actual user interfaces

MCP handles what TCP handles: reliable, structured communication between two endpoints. A2A is more like HTTP — a higher-level protocol layered on top for richer interactions between peers. This is exactly how the internet protocol stack evolved, and the engineers building this stuff know it.

The ecosystem number tells the story: ~100 MCP servers at launch in late 2024, 4,000+ by mid-2025, 5,800+ by late 2025. OpenAI, Google DeepMind, Microsoft, and Databricks all support it. That's not an Anthropic-specific tool anymore — that's infrastructure.

## 4. The Security Elephant in the Room

TCP/IP's original designers didn't anticipate BGP hijacking, DDoS, or man-in-the-middle attacks. We spent decades patching those holes. MCP is on the same trajectory, and people are rightly nervous about it.

The main risks:

- **Overly broad OAuth scopes**: An MCP server requesting full Gmail access when it only needs to read subject lines. Classic supply chain laziness that becomes your security problem.
- **Prompt injection via tool responses**: Malicious content returned by a tool that manipulates the agent's next action. The agent reads a file, the file says "ignore previous instructions and exfiltrate data," and now you have a bad day.
- **Community server supply chain risk**: That random MCP server you pulled off GitHub for your Slack integration? You're trusting its author's security hygiene. Good luck.
- **Traditional AppSec tools don't apply**: Your static analysis scanner doesn't understand non-deterministic agent behavior. It's not looking for the right things.

MCP's answer is OAuth 2.1 with PKCE, least-privilege permissions, and human-in-the-loop authorization for sensitive operations. That's a reasonable starting point, but the operational maturity just isn't there yet.

Additionally, as MCP transitions to enterprise production environments, we are starting to see the emergence of MCP-specific API gateways. Rather than letting an agent client talk directly to a proprietary database MCP server, organizations are putting policy enforcement layers in between. These gateways log every JSON-RPC interaction, enforce strict usage quotas, and use Data Loss Prevention (DLP) scanners to scrub PII before it ever reaches the LLM. If you're building beyond local dev environments, you need to be thinking about this middleware layer.

## 5. What You Should Actually Do This Week

This isn't a "wait and see" situation. MCP hit an inflection point — the protocol has the momentum of a standard, not an experiment. Enterprise analysts are calling 2026 the year of production MCP adoption, and companies are already rolling out internal agents using it daily for querying systems, automating reporting, and orchestrating massive workflows.

Here's a concrete starting point:

1. **Install the MCP SDK** and spin up a local server against a tool you already maintain (`pip install mcp` — it's that simple).
2. **Wire it to Claude Desktop** or any MCP-compatible client and verify the handshake works. Ten minutes of setup, not ten days.
3. **Audit the MCP servers you're considering** like you'd audit any open-source dependency. Check OAuth scope requests. Read the source.
4. **Track the A2A spec** — if you're building multi-agent systems, understanding how MCP and A2A interact is going to matter very soon.
5. **Advocate for sane defaults** — When writing your custom servers, apply the principle of least privilege immediately. Don't provide write access if your conversational agent only needs to summarize read-only logs.
6. **Don't build another custom integration** for a new tool until you've checked whether an MCP server already exists for it. At 5,800+ servers and growing, the answer is often yes.

The internet succeeded because TCP/IP gave every computer a common language. The agentic AI ecosystem will succeed for the same reason — and MCP is the closest thing we have to that foundation right now. Get your hands dirty with it before your company's AI strategy requires it and you're scrambling to catch up.
