# Day 8 — API Gateway & Agent-Facing APIs

Topic for today: **API Gateway + agent-facing API design**. We've shipped Lambda, MCP, and a containerized agent — time to put a proper API in front so the gap between "works on my machine" and "works for users" closes.

## Topics

1. API Gateway — REST vs HTTP API, when to pick which, throttling and usage plans
2. Custom domains, mTLS, and Cognito / JWT authorizers
3. Streaming agent responses — SSE vs WebSockets, the 29s API GW timeout gotcha
4. MCP-over-HTTP — exposing an MCP server with auth, scopes, and rate limits
5. OpenAPI for agent tools — schema-first design so the LLM gets clean tool descriptions

## Notes

- [punith.md](./punith.md) — Agentic AI angle on the topics
- [chandana.md](./chandana.md) — AWS angle on the topics

Tracking issue: [#11](https://github.com/PunithVT/CSvsPVT/issues/11)
