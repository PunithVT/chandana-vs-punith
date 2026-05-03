# Day 4 — MCP (Model Context Protocol)

Topic for today: **Model Context Protocol (MCP)**. Picking MCP because it's the protocol where Chandana's AWS work and my agent work actually meet — feels like the right thing to learn together.

## Topics

1. What MCP is and why it matters — client / server / transport, how it differs from plain function-calling
2. Building a Python MCP server — exposing tools, resources, and prompts
3. Connecting an MCP server to Claude Desktop / IDE clients (stdio vs SSE transport)
4. Hosting MCP servers on AWS — Lambda for short tools vs Fargate for long-running ones
5. Securing MCP — auth, scopes, rate-limiting, handling secrets the server needs

## Notes

- [punith.md](./punith.md) — Agentic AI angle on the topics
- [chandana.md](./chandana.md) — AWS angle on the topics

Tracking issue: [#4](https://github.com/PunithVT/chandana-vs-punith/issues/4)
