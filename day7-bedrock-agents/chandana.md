# Day 7 — Chandana (AWS)

Notes, labs, and experiments for Day 7.

Picking up from the [Day 7 topics](./README.md) — focusing on the AWS-heavy ones:

- [ ] Bedrock Agents core model — agent, action groups, knowledge bases, sessions
- [ ] Setting up a Bedrock Knowledge Base — S3 source, OpenSearch Serverless backing, ingestion job
- [ ] Action groups — function schema vs OpenAPI, the Lambda contract Bedrock expects
- [ ] Agent prompt strategy — system prompt, instructions, advanced prompts override
- [ ] Tracing + debugging Bedrock Agents — agent traces, model invocation logs, when the agent loops

## Notes

_to be filled in as I go_

# Day 7 — AWS Bedrock

Notes covering Bedrock Agents, Knowledge Bases, Action Groups, Prompt Strategy, and Tracing.

---

## What is Amazon Bedrock?

Bedrock is a fully managed AWS service that gives you API access to foundation models — Claude, Llama, Mistral, Titan, Cohere — without managing any servers or GPUs. You send a prompt, you get a response. Everything in between (compute, scaling, model updates) is AWS's problem.

Beyond raw model calls, Bedrock gives you four things:

| Capability | What it does |
|---|---|
| **Foundation Model API** | Direct access to models via a single unified API |
| **Agents** | Autonomous loops that plan, call your tools, and complete multi-step tasks |
| **Knowledge Bases** | Managed RAG — lets the model search your documents before answering |
| **Custom Models** | Fine-tune a foundation model on your own data |

---

## Topic 1 — Bedrock Agents Core Model

### What an Agent is

A regular model call is simple: you send text, you get text back. The model doesn't *do* anything — it just generates a response.

An Agent is different. It can take action. You give it a goal (written in plain English), a set of tools (your Lambda functions), and optionally a set of documents to look things up in. When a user sends a message, the agent decides on its own which tools to call, in what order, and with what parameters — and it keeps going until it has a complete answer.

### The four components

| Component | Role |
|---|---|
| **Agent** | The brain — a foundation model plus your instructions |
| **Action Groups** | The hands — Lambda functions the agent can invoke |
| **Knowledge Base** | The memory — documents the agent can search |
| **Sessions** | The conversation — maintains context across multiple turns |

### How it thinks — the ReAct loop

Bedrock uses the ReAct pattern: **Reason → Act → Observe → repeat**.

Each turn works like this:

1. **User sends a message** — "What are the open P1 tickets for ERP?"
2. **Thought** — The agent reasons: "I need to call `list_tickets` with severity=P1"
3. **Action** — It calls your Lambda with those parameters
4. **Observation** — Your Lambda returns the ticket data
5. **Thought** — "I have enough to answer now"
6. **Final answer** — The agent responds to the user

If the observation is incomplete, the agent goes around the loop again — calling another tool or the same tool with different parameters. By default Bedrock allows up to **10 orchestration steps** before it stops and returns an error.

### Sessions

A session is how the agent remembers the conversation. You pass a `sessionId` (a string you define, typically per-user) on every `invoke_agent` call. As long as the session ID is the same, the agent has memory of what was said before. Sessions expire after a period of inactivity.

---

## Topic 2 — Setting Up a Knowledge Base

### What it is

A Knowledge Base (KB) is managed RAG. RAG stands for Retrieval-Augmented Generation — the idea is to store your facts in a searchable database and inject the relevant pieces into the prompt at query time, rather than asking the model to remember things it was never trained on.

### The ingestion pipeline

When you set up a KB, Bedrock runs your documents through a pipeline before they're searchable:

**S3 → Chunk → Embed → Store in OpenSearch Serverless**

| Stage | What happens |
|---|---|
| **S3 (source)** | You upload PDFs, DOCX, HTML, TXT, MD files to an S3 bucket |
| **Chunking** | Bedrock splits each document into small overlapping pieces — default is ~300 tokens with 20% overlap |
| **Embedding** | Each chunk is converted into a vector (a list of ~1536 numbers that captures its meaning) using Amazon Titan Embeddings |
| **Vector store** | Vectors are stored in OpenSearch Serverless (AOSS), which supports fast similarity search |

### How retrieval works at query time

When a user asks a question:

1. The question is also converted into a vector (embedded)
2. Bedrock finds the chunks whose vectors are closest to the question vector — this is semantic similarity, not keyword matching
3. Those chunks are injected into the model's prompt as context
4. The model answers using the retrieved context, not from memory

### Critical gotcha — syncing

**Uploading files to S3 does not automatically ingest them.** You must explicitly trigger a sync job — either by clicking "Sync" in the console or calling `start_ingestion_job` from the AWS SDK. Until the job completes, new files don't exist as far as the KB is concerned.

### Cost warning — OpenSearch Serverless

AOSS has a minimum charge of roughly **$700/month** for two OCUs, even with zero traffic. For a low-volume platform like Rooman, consider using **Aurora PostgreSQL with pgvector** or **Pinecone** as the vector store instead — both are supported by Bedrock and are significantly cheaper at small scale.

---

## Topic 3 — Action Groups

### What they are

An Action Group is a set of operations the agent can invoke — your APIs, database queries, deployment triggers, whatever. Under the hood it's a Lambda function. You describe the available operations in a schema, and Bedrock uses that schema to decide when and how to call your Lambda.

### Two ways to define the schema

**Option A — Inline function schema**

You describe the function name and its parameters directly in the Bedrock console or CDK. This is simpler and doesn't require a separate file. Good for small action groups with a few operations.

**Option B — OpenAPI spec (stored in S3)**

A full OpenAPI 3.0 YAML file uploaded to S3 and referenced in the action group config. Supports complex types, multiple operations, and reusable components. Required for real production APIs.

The choice between them is mostly about complexity — start with inline, move to OpenAPI when you outgrow it.

### The Lambda contract

Bedrock sends a specific JSON payload to your Lambda and expects a specific JSON shape back. Getting this wrong is the most common reason action groups silently fail.

**What Bedrock sends your Lambda:**
- Which action group was called
- Which function/operation was invoked
- The parameter names and values the agent decided to use
- Session attributes (any state you've stored on the session)

**What you must return:**
- A response body with a `TEXT` key containing your data as a string
- Wrapped in a specific envelope structure Bedrock expects

If your response doesn't match the expected shape, Bedrock treats it as a failed tool call. The agent may retry or give up.

### Resource policy — easy to miss

You must manually grant Bedrock permission to invoke your Lambda by adding a resource-based policy to the function. Bedrock doesn't do this automatically. The error when it's missing is not obviously "missing permission" — it looks like the action group simply isn't being called.

---

## Topic 4 — Agent Prompt Strategy

### How the prompt is assembled

The final prompt the model sees is built from multiple sources layered on top of each other. You control each layer separately.

### Layer 1 — Agent instructions (most important)

This is the plain-English text you write in the "Instructions for the Agent" field in the Bedrock console. It's the most important thing you configure. It defines:

- What the agent is and what it's for
- What it should and should not do
- How it should respond (format, tone, what to do when unsure)

**Be specific. Negative constraints matter as much as positive ones.**

Good: *"You may NOT restart any service without explicit user confirmation. If the user asks you to fix something, explain what the fix would be but do not execute it."*

Bad: *"You are a helpful AWS assistant."*

### Layer 2 — Orchestration template (usually leave this alone)

This is Bedrock's internal template that structures each reasoning step — the Thought/Action/Observation XML blocks the model uses internally. Bedrock has an optimised default. Don't touch this unless you have a very specific reason.

### Layer 3 — Advanced prompt overrides (use sparingly)

Bedrock lets you override individual prompt stages:

| Stage | What it controls |
|---|---|
| **Pre-processing** | How the user's input is interpreted before reasoning begins |
| **Orchestration** | The reasoning and tool-calling loop |
| **KB response generation** | How retrieved document chunks are summarised |
| **Post-processing** | The final response before it reaches the user |

The most common legitimate use of overrides is **post-processing** — for example, stripping XML tags from the output or enforcing a specific response format. Overriding the orchestration stage is risky: if you break the ReAct format, the agent stops calling tools correctly.

---

## Topic 5 — Tracing and Debugging Bedrock Agents

### Why tracing matters

Agents fail in opaque ways. The model might loop until it hits the step limit, call the wrong tool, pass incorrect parameters, or ignore a tool entirely — and without tracing enabled, you get no visibility into why. The final response just looks wrong.

### How to enable tracing

Pass `enableTrace=True` on every `invoke_agent` call during development. The response is a streaming event — you iterate through it, and trace events come alongside the answer chunks. Each trace event shows one step of the ReAct loop: the model's reasoning, what tool it decided to call, what parameters it used, and what your Lambda returned.

### CloudWatch logging for production

Enable model invocation logging at the account level. This sends every prompt, completion, and tool call to a CloudWatch Logs group — so you can debug issues after the fact without having to reproduce them in dev.

### Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Agent loops until step limit | Lambda returning data the model can't parse, or no clear stopping condition in instructions | Fix Lambda response format; add explicit stopping criteria in instructions |
| Agent ignores the tool | Action group description doesn't match how users phrase requests | Rewrite the action group description to match natural phrasing |
| Wrong parameters sent to Lambda | Parameter descriptions are ambiguous | Be explicit — "must be a UUID string like 'abc-123'", not just "the ID" |
| KB returns irrelevant chunks | Data not re-synced after upload; chunk size too large | Run ingestion job; reduce chunk size |
| ValidationException on invoke | Config changed but agent not re-prepared | Call `prepare_agent()` and redeploy alias |

### The prepare_agent gotcha

Any time you edit the agent — instructions, action groups, knowledge base, model — you must call `prepare_agent()` before the changes take effect. Then you need to deploy a new alias (or update the existing one) to make the prepared version live. Forgetting this is the single most common reason "my change didn't work."

---

*Day 7 — AWS Bedrock notes. Part of the ongoing AWS + DevOps study series.*