# Day 4 — Punith (Agentic AI)

Notes, code, and experiments for Day 4.

Picking up from the [Day 4 topics](./README.md) — focusing on the protocol and server-side pieces:

- [x] What MCP is and why it matters — client / server / transport, how it differs from plain function-calling
- [x] Building a Python MCP server — exposing tools, resources, and prompts
- [x] Connecting an MCP server to Claude Desktop / IDE clients (stdio vs SSE transport)

## Notes

# MCP (Model Context Protocol) — Complete Notes

If you've ever wired LangChain tools to Claude, then to GPT, then to a custom agent, then watched yourself rewrite the same integration three times — MCP is the answer to that. It's a **standard protocol** for connecting LLMs to external tools and data, so the same server works with any client that speaks MCP.

The mental model:

```
Before MCP:
  every agent ──→ every tool integration is custom-glued

After MCP:
  every MCP-compatible agent  ←──→  every MCP server
                       (one protocol, plug-and-play)
```

People call MCP "the USB-C of AI integrations" for exactly this reason. One cable, many devices.

---

# CONCEPT 1 — What MCP Is

---

## The One-Sentence Intuition

> MCP is an **open protocol** that lets any LLM client talk to any tool server through a single, standard interface — so you write the integration once and it works everywhere.

Anthropic introduced MCP in late 2024. By 2025 it had been adopted by Claude Desktop, Cursor, Cline, Continue, and a long tail of IDE and agent frameworks. By early 2026 there are open-source MCP servers for GitHub, Slack, Postgres, AWS, Notion — most of the things you'd want to wire up.

---

## The Architecture — Client, Server, Transport

```
┌────────────────────┐   MCP protocol    ┌────────────────────┐
│  CLIENT            │ ◄────────────────►│  SERVER            │
│  (Claude Desktop,  │                   │  (your code —      │
│  Cursor, Cline,    │                   │  exposes tools,    │
│  custom agent)     │                   │  resources,        │
│                    │                   │  prompts)          │
└────────────────────┘                   └────────────────────┘
         ▲                                         ▲
         │                                         │
    LLM lives here                          External world
   (it picks tools to                    (DB, API, files, AWS,
    call based on user                    whatever your tools
    input + tool descs)                   wrap)
```

**Three roles:**

| Role        | What it is                                                          |
|-------------|---------------------------------------------------------------------|
| **Host**    | The application running the LLM (Claude Desktop, Cursor, etc.)      |
| **Client**  | The MCP client library inside the host that talks to servers        |
| **Server**  | A standalone process that exposes tools, resources, or prompts      |

A single host can connect to many servers at once. Each server runs as its own process — sandboxed, independently restartable, swappable.

---

## The Three Primitives — Tools, Resources, Prompts

Everything an MCP server exposes is one of three things:

| Primitive | What it is                              | When to use                                  |
|-----------|------------------------------------------|----------------------------------------------|
| **Tool**     | A function the model can decide to call  | Any action — DB query, API call, file write |
| **Resource** | Read-only data the host can attach as context | Documents, logs, schemas — things the model *reads* |
| **Prompt**   | A reusable prompt template the user can invoke | "Summarize this in our house style", canned workflows |

```
Tool      → "do something"     (model-driven, per-call)
Resource  → "here is data"     (host-driven, attached to context)
Prompt    → "use this prompt"  (user-driven, invoked by name)
```

Most MCP servers in the wild are tool-only. Resources and prompts are powerful but less commonly used.

---

## Transport — How Client and Server Talk

The MCP protocol itself is **JSON-RPC 2.0** over a transport. The two transports that matter:

```
stdio (standard input / output)
  Server is a child process of the host
  Host writes JSON-RPC messages to server's stdin
  Server writes responses to its stdout
  Use for: local servers, Claude Desktop, IDE plugins

SSE / Streamable HTTP (Server-Sent Events)
  Server runs as a long-lived HTTP service
  Host connects over HTTPS, holds the connection open
  Use for: remote / cloud-hosted servers, multi-user setups
```

| Transport          | Best for                          | Latency  | Auth           |
|--------------------|-----------------------------------|----------|----------------|
| **stdio**          | Local tools, single-user, IDE     | Lowest   | None needed    |
| **SSE / HTTP**     | Remote / shared / cloud servers   | Network-bound | Bearer / OAuth / mTLS |

> Default to **stdio** while learning. It's simpler, doesn't need a network, doesn't need auth. Move to SSE only when you need to host the server on AWS or share it across machines.

---

## How MCP Differs From Plain Function-Calling

If you've used OpenAI's function-calling or Anthropic's tool-use API, you know the pattern: the LLM picks a function, the host runs it, the result goes back to the LLM. So why MCP?

| Aspect                     | Plain function-calling          | MCP                                       |
|----------------------------|----------------------------------|-------------------------------------------|
| Where tools live           | In the host's code               | In separate server processes              |
| Adding a new tool          | Redeploy the host                | Start another server, register it          |
| Sharing tools across hosts | Copy-paste the integration       | One server, any MCP-compatible host        |
| Lifecycle                  | Same as host                     | Independent — restart, version, sandbox    |
| Discovery                  | Hardcoded                        | `tools/list` returns the catalog           |
| Streaming results          | Provider-specific                | Built into the protocol                    |

> The big idea: function-calling is an **API** between *one* model and *one* host. MCP is a **protocol** between *any* model and *any* server. That's the whole upgrade.

---

# CONCEPT 2 — Building a Python MCP Server

---

## The Python SDK

The official package is `mcp`. It ships two layers:

- **Low-level `Server`** — full control over the JSON-RPC handshake, hand-write each handler.
- **High-level `FastMCP`** — decorator-based, hides the protocol, easiest start.

`FastMCP` is what you'll use 95% of the time. It feels like writing FastAPI for the LLM.

```bash
pip install mcp
```

---

## A Minimal Working Server (FastMCP, stdio)

```python
# server.py
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("rooman-tools")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two integers and return the sum."""
    return a + b

if __name__ == "__main__":
    mcp.run()   # defaults to stdio transport
```

Running this directly does nothing — there's no human interface. The way you *use* it is by registering it with an MCP host (Claude Desktop, Cursor, etc.) — they spawn it as a subprocess and start sending JSON-RPC over stdio. We'll cover that in Concept 3.

---

## Tools — The Workhorse

A tool is a Python function with a typed signature and a docstring. The decorator turns it into something the LLM can invoke.

```python
@mcp.tool()
def search_crm(query: str, limit: int = 10) -> list[dict]:
    """Search the Rooman CRM for leads matching a query.

    Args:
        query: Free-text search query.
        limit: Max results to return (default 10).

    Returns:
        List of {id, name, email, status} dicts.
    """
    return crm_client.search(query, limit=limit)
```

The model sees:
- **Name** — `search_crm` (verb-style is best)
- **Description** — the docstring (this is the most important thing — the model picks tools based on it)
- **Parameter schema** — generated from the type hints
- **Return type** — generated from the type hints

### Tool design rules that come up over and over

| Rule                                     | Why                                                |
|-------------------------------------------|----------------------------------------------------|
| Verb-style name (`search_crm`, not `crm`) | Models match by name first, description second     |
| Detailed docstring with *examples*        | Becomes the description the model reads            |
| Strict typed parameters                   | Models hallucinate args; types reduce slop         |
| Return JSON-serializable data             | Next agent step parses it; prose causes drift      |
| Validate inputs in the function           | Models lie about types; never trust them           |
| Keep tools small + composable             | A "do everything" tool gets used randomly          |

> If the model keeps misusing a tool, the fix is almost always a clearer **docstring**, not a smarter model. Treat docstrings as production code.

---

## Tools That Return Errors Cleanly

A tool that raises an exception sends back a JSON-RPC error. The model sees a generic failure — not great. Better: catch expected errors and return structured info the model can reason about.

```python
@mcp.tool()
def get_user(user_id: str) -> dict:
    """Look up a user by ID. Returns user dict or {error: "not_found"}."""
    user = db.find_one(user_id)
    if not user:
        return {"error": "not_found", "user_id": user_id}
    return {"user": user}
```

This way the LLM can decide to ask the user for a different ID, instead of just saying "the tool failed."

---

## Resources — Read-Only Context

Resources expose data the host can fetch and inject into context. Think of them as "files" the LLM can read.

```python
@mcp.resource("crm://leads/{lead_id}")
def lead_profile(lead_id: str) -> str:
    """Markdown summary of a single lead."""
    lead = db.get_lead(lead_id)
    return f"# {lead.name}\n\n- Email: {lead.email}\n- Status: {lead.status}\n"
```

Resources have **URIs** (custom scheme). The host enumerates them, the user (or the model) picks one, and the host fetches it via `resources/read`.

| Use resources for           | Don't use resources for                 |
|------------------------------|------------------------------------------|
| Documents, logs, schemas     | Anything mutating state                  |
| Static catalogs              | Anything time-sensitive that needs fresh data |
| Files in the project         | Things that should be model-callable (use tools) |

> Resources are *read*. Tools are *do*. If a primitive could mutate anything, it's a tool.

---

## Prompts — Reusable Templates

Prompts let you ship canned workflows the user can invoke by name from the host UI.

```python
@mcp.prompt()
def summarize_lead(lead_id: str) -> str:
    """Generate a summary prompt for a specific lead."""
    return (
        f"Pull lead {lead_id} from the CRM and summarize their status, "
        f"recent activity, and what should happen next."
    )
```

In Claude Desktop, this shows up as a slash command (`/summarize_lead`) the user can run. Useful for canned playbooks where you don't trust the user to phrase the prompt well.

---

## Logging and Debugging

stdio servers can't `print()` to stdout — that's where the JSON-RPC messages go. Anything you print to stdout corrupts the protocol.

**Always log to stderr instead:**

```python
import sys, logging

logging.basicConfig(stream=sys.stderr, level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(message)s")
logger = logging.getLogger("mcp-server")

@mcp.tool()
def search_crm(query: str) -> list[dict]:
    """Search the CRM."""
    logger.info(f"search_crm called with query={query!r}")
    ...
```

Claude Desktop captures stderr to a log file (`~/Library/Logs/Claude/mcp.log` on macOS) — that's where you go when something breaks.

> Print to stdout = silently broken server. The host disconnects with no visible error. This is the single most common MCP gotcha.

---

## Validating Inputs (the security mindset)

LLMs hallucinate arguments. Validate every input as if it came from a public API.

```python
import re

@mcp.tool()
def get_user(user_id: str) -> dict:
    """Fetch user by ID. ID must be a UUID."""
    if not re.fullmatch(r"[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}", user_id):
        return {"error": "invalid_user_id_format"}
    return db.get_user(user_id)
```

Day 2's IAM lessons apply here in full — see [day2-iam/punith.md](../day2-iam/punith.md) for the full agent-side security playbook (prompt injection, tool authorization, scoped credentials).

---

# CONCEPT 3 — Connecting an MCP Server to Clients

---

## Claude Desktop — The Default Client

Claude Desktop is the easiest place to test an MCP server. It launches your server as a stdio child process based on a config file.

### macOS / Linux config path

```
~/Library/Application Support/Claude/claude_desktop_config.json   (macOS)
~/.config/Claude/claude_desktop_config.json                        (Linux)
```

### Windows

```
%APPDATA%\Claude\claude_desktop_config.json
```

### Minimal config

```json
{
  "mcpServers": {
    "rooman-tools": {
      "command": "python",
      "args": ["/absolute/path/to/server.py"]
    }
  }
}
```

After saving, **restart Claude Desktop** (quit + reopen, not just close the window). Claude reads the config only at startup.

### With a virtualenv

```json
{
  "mcpServers": {
    "rooman-tools": {
      "command": "/Users/punith/projects/mcp/.venv/bin/python",
      "args": ["/Users/punith/projects/mcp/server.py"]
    }
  }
}
```

### With env vars

```json
{
  "mcpServers": {
    "rooman-tools": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "env": {
        "DATABASE_URL": "postgres://...",
        "API_KEY": "..."
      }
    }
  }
}
```

> Don't put real secrets in the JSON config — it's plaintext on disk. Read them from a keychain or `.env` file inside the server instead.

---

## Verifying It Works

Once the config is in place and Claude Desktop has restarted:

1. The MCP icon (a small plug / hammer) appears in the chat input.
2. Click it — your server name (`rooman-tools`) should be listed with its tools.
3. Ask Claude something that should trigger a tool: "search the CRM for Acme."

If nothing happens:

| Symptom                          | Likely cause                                  |
|----------------------------------|-----------------------------------------------|
| Server name not visible          | Config path wrong, or JSON is malformed       |
| Tools not listed                 | Server crashed at startup (check stderr log)  |
| Tool called but errors           | Working directory / env / dependency issue    |
| Tool seems ignored by model      | Docstring isn't clear enough                  |

The MCP log file is your best friend:

```
macOS:   ~/Library/Logs/Claude/mcp.log
         ~/Library/Logs/Claude/mcp-server-rooman-tools.log
Linux:   ~/.config/Claude/logs/...
```

---

## The MCP Inspector — Standalone Debugger

Anthropic ships a standalone debugger (`@modelcontextprotocol/inspector`). It runs your server in isolation so you can poke it without involving Claude Desktop:

```bash
npx @modelcontextprotocol/inspector python /path/to/server.py
```

Opens a local web UI where you can:
- See all tools / resources / prompts
- Call tools with custom arguments
- Watch the JSON-RPC traffic live

This is a much faster debugging loop than restart-Desktop-every-time.

---

## IDE Clients — Cursor, Cline, Continue

Most IDE-side AI tools have adopted MCP. They follow the same pattern as Claude Desktop — a JSON config that points to your server.

**Cursor** — Settings → MCP, drop a server entry.

**Cline (VS Code)** — `cline_mcp_settings.json` in the extension's storage dir.

**Continue (VS Code / JetBrains)** — `~/.continue/config.json`, `mcpServers` section.

The shape of the config is essentially identical across hosts. Once you've got one server working in Claude Desktop, getting it into an IDE is config copy-paste.

---

## SSE / Streamable HTTP — When You Outgrow stdio

stdio runs the server on the user's machine. That's fine for personal tools, but breaks down when:

- The server needs cloud-side compute or data
- Multiple users should share a single server
- You want to host the server on AWS Lambda or Fargate
- The server needs to outlive any single client session

For those cases, MCP supports a remote transport over HTTPS using Server-Sent Events (SSE).

```python
# server.py — SSE
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("rooman-tools")

@mcp.tool()
def search_crm(query: str) -> list[dict]:
    """Search the CRM."""
    return crm_client.search(query)

if __name__ == "__main__":
    mcp.run(transport="sse")   # binds an HTTP server
```

Client config switches from `command` + `args` to a URL:

```json
{
  "mcpServers": {
    "rooman-tools": {
      "url": "https://mcp.rooman.com/sse",
      "headers": { "Authorization": "Bearer ..." }
    }
  }
}
```

Hosting on AWS (Lambda Function URL vs Fargate ALB), and the auth / scope / rate-limit layers that go with remote MCP, are covered in detail in [chandana.md](./chandana.md) — that's the AWS angle for today.

> If you're using API Gateway in front of an SSE server, watch the **29-second timeout** — it'll silently kill long-lived MCP connections. Use a Lambda Function URL or an ALB instead.

---

## Common Mistakes

| Mistake                                          | Symptom                                         |
|--------------------------------------------------|-------------------------------------------------|
| `print()` to stdout in a stdio server            | Host disconnects, no visible error              |
| Relative path to `server.py` in config           | Server fails to start — host launches in `/`    |
| Forgetting to restart Claude Desktop after config| Config seems ignored                            |
| Vague docstrings (`"Get data"`)                  | Model never picks the tool                      |
| Tool returns a Python object directly            | Serialization error; tool appears broken        |
| Long-running tool with no progress signal        | Host UI looks frozen; user cancels              |
| Same tool name across two servers                | Host picks one arbitrarily; the other is shadowed |
| Real secrets in `claude_desktop_config.json`     | Plaintext on disk, in shell history if copied   |
| API Gateway in front of an SSE server            | 29s timeout kills the connection                |

---

# Quick Reference — All Three Concepts

```
WHAT MCP IS
  Open protocol for LLM ↔ tool integration (Anthropic, late 2024)
  JSON-RPC 2.0 over stdio or SSE/HTTPS
  "USB-C of AI integrations" — write the integration once
  Three roles: Host (Claude Desktop) / Client / Server (your code)
  Three primitives: Tools (do), Resources (read), Prompts (templates)

TRANSPORT — STDIO vs SSE
  stdio  → local, child process, lowest latency, no auth needed
  SSE    → remote, long-lived HTTP, needs auth, can be hosted on AWS
  Default to stdio while learning; move to SSE only for shared / cloud servers

MCP vs PLAIN FUNCTION-CALLING
  Function-calling: tools live in the host, redeploy to add one
  MCP: tools live in separate server processes, plug-and-play
  Same MCP server works with Claude / Cursor / Cline / custom agents

PYTHON SDK — FastMCP
  pip install mcp
  from mcp.server.fastmcp import FastMCP
  mcp = FastMCP("server-name")
  Decorators: @mcp.tool(), @mcp.resource(), @mcp.prompt()
  mcp.run() defaults to stdio; mcp.run(transport="sse") for remote

TOOL DESIGN
  Verb-style name, typed args, JSON-serializable return
  Docstring = the LLM's view of the tool — write it like docs
  Validate inputs (LLMs hallucinate args)
  Return structured errors instead of raising on expected failures
  Small, composable tools beat one mega-tool

LOGGING — STDIO GOTCHA
  NEVER print() to stdout in a stdio server (corrupts JSON-RPC)
  Use logging with stream=sys.stderr instead
  Claude Desktop captures stderr to ~/Library/Logs/Claude/mcp-*.log

CLAUDE DESKTOP CONFIG
  ~/Library/Application Support/Claude/claude_desktop_config.json (macOS)
  ~/.config/Claude/claude_desktop_config.json                      (Linux)
  %APPDATA%\Claude\claude_desktop_config.json                       (Windows)

  {
    "mcpServers": {
      "name": { "command": "python", "args": ["/abs/path/server.py"] }
    }
  }

  Always absolute paths — host launches from /
  Restart Claude Desktop after every config change

DEBUGGING
  npx @modelcontextprotocol/inspector python server.py
  → standalone web UI, no Claude Desktop needed, sees JSON-RPC live
  Single fastest way to iterate on tool design

IDE CLIENTS
  Cursor / Cline / Continue all support MCP
  Same shape of config — only the file path differs
  Get one server working in Claude Desktop, then config copy-paste

REMOTE HOSTING (SSE)
  mcp.run(transport="sse") binds an HTTP server
  Client config uses URL + Authorization header instead of command/args
  AWS hosting → Lambda Function URL (short tools) or Fargate (long-lived SSE)
  Avoid API Gateway in front of SSE — 29s timeout kills connections

COMMON MISTAKES
  print() to stdout in stdio server
  Relative paths in config (host launches from /)
  Forgetting to restart Claude Desktop
  Vague docstrings → model never picks the tool
  Returning non-JSON Python objects
  Plaintext secrets in claude_desktop_config.json
  Same tool name across two servers (host picks one arbitrarily)
  API Gateway in front of SSE (29s hard timeout)
```
