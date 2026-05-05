# chandana-vs-punith 

> A **30-day learning challenge** where two engineers go deep on two of the most in-demand stacks of 2026 — **AWS** and **Agentic AI** — and document everything publicly, including the beginner-to-advanced mistakes most tutorials skip.

**Punith** → Agentic AI track (Claude, Amazon Bedrock, MCP, LangChain, RAG)
**Chandana** → AWS track (Lambda, IAM, ECS, EKS, OpenSearch, CloudWatch)

Each day we both write up the same topic from our own angle. The goal: by Day 30, both of us are comfortable shipping production-grade work that crosses both domains — agents that run on AWS, AWS systems that integrate with agents.

## Why this repo exists

Most "learn AWS" or "learn AI agents" tutorials show you the happy path. They don't tell you that:

- A psycopg2 wheel built on macOS will silently fail on Lambda
- A Bedrock Agent's `parameters` field is a list of `{name, value}` objects, not a dict
- A `Co-authored-by` trailer is the difference between one contributor and two on a GitHub commit
- An MCP server's tool description is what the model actually reads — your code matters less than that string

This repo is a **public, daily learning log** of those gotchas. Plain language, working code, real mistakes.

## What you'll find here

- **Daily long-form notes** under each `dayN-<topic>/` folder — written so a beginner can follow along but useful enough that experienced engineers find new gotchas.
- **Two angles per topic** — `chandana.md` (AWS implementation) and `punith.md` (Agentic AI implementation), so the same concept is shown from both sides.
- **Production-aware examples** — every code snippet considers cost, IAM scope, cold starts, idempotency, and security, not just "hello world".
- **Tracking issues** — each day has a GitHub issue with the topic plan; closed when the day's notes are merged.

## Topics covered (Day 1 → Day 6)

| Day | Topic | Folder | Issue |
| --- | --- | --- | --- |
| 1 | AWS Lambda & Python Lambda — handler, event, context, triggers, agent-tool patterns | [day1-lambda/](./day1-lambda/) | [#1](https://github.com/PunithVT/chandana-vs-punith/issues/1) |
| 2 | IAM & Security — policies, AssumeRole, prompt injection, secrets handling | [day2-iam/](./day2-iam/) | [#2](https://github.com/PunithVT/chandana-vs-punith/issues/2) |
| 3 | RAG end-to-end — embeddings, chunking, OpenSearch, Bedrock Knowledge Bases, reranking | [day3-rag/](./day3-rag/) | [#3](https://github.com/PunithVT/chandana-vs-punith/issues/3) |
| 4 | MCP (Model Context Protocol) — building servers, hosting on AWS, Claude Desktop integration | [day4-mcp/](./day4-mcp/) | [#4](https://github.com/PunithVT/chandana-vs-punith/issues/4) |
| 5 | Observability — CloudWatch, X-Ray, LangSmith, OpenTelemetry, agent traces | [day5-observability/](./day5-observability/) | [#5](https://github.com/PunithVT/chandana-vs-punith/issues/5) |
| 6 | Containers — ECS, EKS, Fargate, Dockerized agents, MCP servers in containers | [day6-containers/](./day6-containers/) | [#6](https://github.com/PunithVT/chandana-vs-punith/issues/6) |

Days 7–30 will be planned as we go, based on what gaps show up in our work.

## Folder structure

```
chandana-vs-punith/
├── day1-lambda/
│   ├── README.md     # day's topic + plan
│   ├── chandana.md   # AWS deep-dive
│   └── punith.md     # Agentic AI deep-dive
├── day2-iam/
│   └── ...
└── ...
```

Folders use the convention `day<N>-<short-topic>` so the topic is visible from the repo root without having to click in.

## Tech stack we're touching

**AWS services** — AWS Lambda · IAM · S3 · DynamoDB · OpenSearch Serverless · Amazon Bedrock · Bedrock Knowledge Bases · ECS · EKS · Fargate · API Gateway · EventBridge · SQS · CloudWatch · X-Ray · Secrets Manager · KMS · Step Functions

**Agentic AI** — Claude (Anthropic) · Amazon Bedrock Agents · LangChain · LangGraph · MCP (Model Context Protocol) · RAG (Retrieval-Augmented Generation) · Vector embeddings · Cohere Rerank · Strands Agents · LangSmith · OpenTelemetry

**Python** — boto3 · langchain · anthropic SDK · mcp · fastapi · pydantic · aws-lambda-powertools · moto

## Following along

If you're learning AWS, Agentic AI, or both — this repo is meant to be a useful side-by-side reference, not a course.

- **Star the repo** to follow daily updates as Days 7–30 roll out.
- **Spot something wrong?** Open an issue or PR — corrections welcome, that's the whole point of learning in public.
- **Want the same structure for your own challenge?** Fork it and replace the names. The daily log + dual-angle layout works for any two-track learning.

## Common questions

**Is this a course?**
No. It's a public learning journal. We're learning *as* we write — not teaching from expertise.

**Can I follow only one track?**
Yes. Read only `chandana.md` files for AWS, only `punith.md` files for Agentic AI. The day's `README.md` ties them together.

**What level is this aimed at?**
The notes are written so a beginner can follow with effort, but every day surfaces gotchas an experienced engineer would also learn from.

**Why two people?**
One person learning two things deeply in 30 days isn't realistic. Two people, one each, comparing notes — is.

---

**Started:** April 2026 · **Length:** 30 days · **Tracks:** AWS · Agentic AI
