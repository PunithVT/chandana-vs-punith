# Day 9 — DynamoDB & Agent Memory

Topic for today: **DynamoDB + agent memory patterns**. Agents are stateless by default — production agents need memory. Picking DynamoDB because it's the AWS-native default for agent memory and it forces us to learn single-table design properly.

## Topics

1. DynamoDB core — partition / sort keys, GSIs, eventual vs strong consistency
2. Single-table design — when it's worth the cognitive load, when it isn't
3. Agent memory patterns — short-term (turn buffer), long-term (semantic), episodic (per-session)
4. Storing conversation history at scale — TTL, archival to S3, retrieval shape
5. Costs and access patterns — RCU/WCU vs on-demand, hot partitions, query vs scan

## Notes

- [chandana.md](./chandana.md) — AWS angle on the topics
- [punith.md](./punith.md) — Agentic AI angle on the topics

Tracking issue: [#12](https://github.com/PunithVT/CSvsPVT/issues/12)
