# Day 5 — Punith (Agentic AI)

Notes, code, and experiments for Day 5.

Picking up from the [Day 5 topics](./README.md) — focusing on the agent-side observability stack:

- [x] LangSmith / OpenTelemetry for agents — token, tool-call, and prompt traces
- [x] Cost and latency observability for LLM apps — what to track per request

## Notes

# Agent Observability — Complete Notes

A traditional web-app trace ends at "the request returned 200 in 142 ms." That's not enough for LLM apps. An agent might call a model 6 times, hit 3 tools, retrieve 12 chunks, and burn $0.04 in tokens — and *still* return the wrong answer. To debug it you need to see all of that, in order, per request.

The mental model:

```
Web app:    request → handler → DB → response             (one straight line)
Agent app:  request → LLM ⇄ tools ⇄ retrieval ⇄ LLM ...   (a tree of decisions)

You can't fix what you can't see.
```

These notes are about making that tree visible — what to capture, where to capture it, and which platform to capture it in.

---

# CONCEPT 1 — Why Agent Observability Is Different

---

## What Traditional APM Misses

Datadog, New Relic, CloudWatch — they all capture HTTP timings, error rates, CPU. None of them, out of the box, capture:

- **Tokens** — input / output per call, per model, per request
- **Cost** — dollars per request, per user, per feature
- **Tool calls** — which tool, what args, what result, in what order
- **Retrieval** — what queries were embedded, what chunks came back, with what scores
- **Prompts** — the *exact* string sent to the model (system + user + history)
- **Model identity** — which model + version + temperature was actually used
- **Decision branches** — the agent picked tool A over tool B; why?

Without these, you can see that "the agent took 8 seconds" but not *why* — was it 6s of LLM, 1.5s of tool, 0.5s of retrieval? Was the LLM call slow because of TTFT or because the prompt was 90K tokens?

> Agent observability = APM + LLM-specific signals. You need both, and most platforms only ship one.

---

## The Shape of an Agent Trace

An agent trace is a **tree**, not a list. Every LLM call can spawn tool calls; every tool call can spawn sub-calls.

```
Trace: "summarize the candidate profile for job X"
│
├── span: agent loop (root)
│   ├── span: retriever.search("candidate skills")     (350 ms, 5 chunks)
│   ├── span: LLM call #1 — gpt-4o-mini                (1.2 s, 1850 in / 240 out tok)
│   │   └── decision: call tool `get_candidate(id)`
│   ├── span: tool: get_candidate(id="abc")            (180 ms)
│   ├── span: LLM call #2 — gpt-4o                     (3.4 s, 4200 in / 510 out tok)
│   │   └── decision: final answer
│   └── span: post-process / format output             (5 ms)
└── total: 5.1 s · 6,290 in / 750 out tok · ~$0.0042
```

Every span has a **parent** (which span spawned it) and a **start/end time**. That's how you reconstruct the tree, even though everything ran asynchronously.

> Once you have this tree per request, every debugging question becomes "look at the tree." Latency? See which span dominates. Hallucination? Look at what the model saw. Wrong tool? Look at the decision before it.

---

# CONCEPT 2 — LangSmith

---

## What LangSmith Is

LangSmith is the observability + eval platform from the LangChain team. It's the easiest "just turn it on and start seeing traces" option for agentic apps. SaaS by default; self-hosted is paid.

It captures:
- Every LangChain run (chains, agents, tools, retrievers)
- Token counts and costs (when the SDK reports them)
- Prompt + output for every model call
- Errors with full stack traces
- Datasets and eval runs

> Under the hood, LangSmith is a tracing system specialized for LLM workloads. If you've used Sentry or Datadog APM, the mental model is the same — except every span knows about tokens and prompts.

---

## Five-Minute Setup

```bash
pip install langsmith langchain
```

```python
import os
os.environ["LANGCHAIN_TRACING_V2"]   = "true"
os.environ["LANGCHAIN_API_KEY"]      = "ls__..."
os.environ["LANGCHAIN_PROJECT"]      = "rooman-prod"  # any name; auto-creates
```

That's it for LangChain code. Every chain, agent, retriever, and tool you run is now traced. Open `smith.langchain.com` and the runs show up live.

---

## Tracing Non-LangChain Code (the `@traceable` decorator)

Most real apps have hand-written code outside LangChain — direct OpenAI / Bedrock calls, custom retrievers, etc. Wrap those with `@traceable` and they show up in the same trace tree.

```python
from langsmith import traceable
from openai import OpenAI

client = OpenAI()

@traceable(run_type="llm")
def call_gpt(prompt: str) -> str:
    resp = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    )
    return resp.choices[0].message.content

@traceable(run_type="tool")
def search_crm(query: str) -> list[dict]:
    return crm_client.search(query)

@traceable(run_type="chain")
def agent_loop(user_msg: str) -> str:
    leads = search_crm(user_msg)
    summary = call_gpt(f"Summarize:\n{leads}")
    return summary

agent_loop("find recent React leads")
```

The decorator gives the function a name (its qualified name by default), a type, and automatically captures inputs / outputs / timing.

| `run_type`     | When to use                                           |
|----------------|-------------------------------------------------------|
| `chain`        | Composite logic (agent loop, RAG pipeline)            |
| `llm`          | Direct model API calls                                |
| `tool`         | Function the agent invokes (search, send_email, etc.) |
| `retriever`    | Vector search / hybrid retrieval                      |
| `parser`       | Output parsing, structured extraction                 |

> Use the right `run_type`. LangSmith UI groups and styles runs by type — getting it right makes the trace much easier to read.

---

## Capturing Tokens and Cost

LangChain's chat models report tokens automatically when the provider does. For raw API calls, attach metadata yourself:

```python
@traceable(run_type="llm")
def call_gpt(prompt: str):
    resp = client.chat.completions.create(model="gpt-4o-mini",
                                           messages=[{"role": "user", "content": prompt}])
    msg = resp.choices[0].message.content

    # Tell LangSmith about token usage explicitly
    from langsmith import get_current_run_tree
    rt = get_current_run_tree()
    if rt:
        rt.add_metadata({
            "input_tokens":  resp.usage.prompt_tokens,
            "output_tokens": resp.usage.completion_tokens,
            "model":         resp.model,
        })
    return msg
```

Once metadata is in place, the LangSmith dashboard auto-aggregates by user, by feature, by model — useful for cost attribution.

---

## Datasets + Evals

The piece most people sleep on. LangSmith lets you:

1. Save real production runs as **examples** in a dataset
2. Re-run them against a new model / prompt / agent version
3. Compare outputs (LLM-as-judge or exact match)

This is how you avoid the "we changed the prompt and regressed three things we forgot to check" trap.

```python
from langsmith import Client
client = Client()

# Save a dataset of real queries
dataset = client.create_dataset("rooman-real-queries-2026-04")
client.create_examples(
    dataset_id=dataset.id,
    inputs=[{"q": "find React leads"}, {"q": "send follow-up to Acme"}],
    outputs=[{"a": "..."}, {"a": "..."}],
)

# Run any function against the dataset
from langsmith.evaluation import evaluate
evaluate(agent_loop, data=dataset.name, evaluators=[my_judge])
```

> Treat your eval set as production code. 30 saved examples is enough to start; build a habit of adding the next failure each time you find one.

---

## When LangSmith Is the Right Pick

| Use LangSmith when                                | Avoid when                                       |
|---------------------------------------------------|--------------------------------------------------|
| You're already on LangChain / LangGraph           | You don't want vendor lock-in                    |
| You want zero-config "just works" tracing         | You need multi-tenant SaaS-grade isolation       |
| You want built-in datasets + LLM-as-judge evals   | Compliance bars sending prompts to a third party |
| Your team is < 50 engineers                       | You already have an OTel + Datadog/Grafana stack |

If two of those "avoid" conditions hit, OpenTelemetry is the better choice — covered next.

---

# CONCEPT 3 — OpenTelemetry for Agents

---

## What OTel Is (and Why It Matters Here)

OpenTelemetry is the **vendor-neutral standard** for traces / metrics / logs. The same instrumentation works against Datadog, New Relic, Grafana Tempo, AWS X-Ray, Honeycomb, Jaeger — change one env var, switch backends.

For agents, OTel matters because:
- It's the only telemetry standard your AWS / SRE team will already know
- It plugs into AWS X-Ray and CloudWatch with no extra work
- It avoids LangSmith / LangFuse / Helicone-style lock-in
- It now has **GenAI-specific semantic conventions** (the `gen_ai.*` attributes) — same field names whether you instrument OpenAI, Bedrock, or Anthropic

---

## The OTel Model (One Slide)

```
TRACE      = a single end-to-end request, identified by trace_id
  SPAN     = one unit of work in that trace, identified by span_id + parent_span_id
    ATTRS  = key-value tags on the span (model name, user_id, token count)
    EVENTS = timestamped log lines inside the span
```

A complete OTel agent trace looks the same shape as the LangSmith tree above — same parent/child relationships, just a different UI.

---

## GenAI Semantic Conventions (the `gen_ai.*` attributes)

OpenTelemetry standardised attribute names for GenAI workloads in 2024–25. Stick to these and any OTel-aware backend can render LLM-specific UIs:

| Attribute                      | What it is                                  |
|--------------------------------|---------------------------------------------|
| `gen_ai.system`                | Provider — `openai`, `anthropic`, `bedrock` |
| `gen_ai.request.model`         | Model id — `gpt-4o`, `claude-3-5-sonnet`    |
| `gen_ai.request.temperature`   | Temperature                                 |
| `gen_ai.response.model`        | Model the provider actually served          |
| `gen_ai.usage.input_tokens`    | Prompt tokens consumed                      |
| `gen_ai.usage.output_tokens`   | Completion tokens                           |
| `gen_ai.operation.name`        | `chat`, `embeddings`, `tool_use`            |
| `gen_ai.tool.name`             | Tool name (when applicable)                 |
| `gen_ai.tool.call.id`          | Tool call id from the API                   |

Use these names exactly. Backends that know these conventions — including X-Ray's GenAI views — light up automatically.

---

## Setting Up OTel for a Python Agent

```bash
pip install opentelemetry-api opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-openai \
            opentelemetry-instrumentation-langchain
```

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

# Wire up: traces go to whatever OTLP endpoint is configured
provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

# Auto-instrument OpenAI / LangChain
from opentelemetry.instrumentation.openai import OpenAIInstrumentor
from opentelemetry.instrumentation.langchain import LangchainInstrumentor
OpenAIInstrumentor().instrument()
LangchainInstrumentor().instrument()
```

Configure the exporter via env vars (this is the part your AWS/SRE side will care about):

```bash
export OTEL_SERVICE_NAME=rooman-agent
export OTEL_EXPORTER_OTLP_ENDPOINT=https://otlp.example.com:4318
export OTEL_EXPORTER_OTLP_HEADERS="api-key=..."
```

After this, every OpenAI / LangChain call automatically emits OTel spans with `gen_ai.*` attributes. No code changes anywhere else.

---

## Manual Spans for Custom Tools

Auto-instrumentation covers the popular SDKs. Anything custom — your retriever, your vector DB client, your business logic — needs hand-rolled spans:

```python
from opentelemetry import trace

tracer = trace.get_tracer("rooman.agent")

def search_crm(query: str) -> list[dict]:
    with tracer.start_as_current_span("tool.search_crm") as span:
        span.set_attribute("gen_ai.tool.name", "search_crm")
        span.set_attribute("query", query)

        results = crm_client.search(query)
        span.set_attribute("result_count", len(results))
        return results
```

These spans become children of whatever span is currently active — usually the LLM call that decided to invoke this tool. The trace tree assembles itself.

> Wrap every tool, every retrieval, every external API call. Inside an agent, anything that takes more than a few ms deserves its own span.

---

## Sending OTel Traces to AWS X-Ray

X-Ray accepts OTLP natively via the **AWS Distro for OpenTelemetry (ADOT)** collector. Two paths:

1. **Lambda layer** — add the ADOT layer to a Lambda function, set `AWS_LAMBDA_EXEC_WRAPPER=/opt/otel-instrument`. Auto-injects the collector.
2. **Sidecar / ECS task** — run the ADOT collector container; your agent points OTLP at `http://localhost:4318`.

Once spans land in X-Ray, you get the same trace tree as LangSmith — but now alongside every other AWS service trace (Lambda, DynamoDB, API Gateway). One pane of glass for the whole request.

The AWS-native side (X-Ray, CloudWatch, sampling) is in [chandana.md](./chandana.md). This file just covers what the agent code emits.

---

## When to Combine LangSmith + OTel

You don't have to pick one. A common production setup:

```
LangSmith    → product / prompt-eng iteration loop (datasets, evals, prompt versioning)
OpenTelemetry → ops / SRE loop (latency, error rates, alerts, AWS-stitched traces)
```

They serve different audiences. ML engineers want LangSmith's prompt diff view; SRE wants Grafana percentiles. Both can run from the same code with no conflict — the LangChain SDK supports both at once.

---

# CONCEPT 4 — Cost and Latency Observability for LLM Apps

---

## The Metrics That Actually Matter

Forget "we have observability." If you can't pull these numbers in 30 seconds, you don't:

| Metric                      | Question it answers                              |
|-----------------------------|--------------------------------------------------|
| **Tokens in / out** per request | Is this prompt growing unboundedly?           |
| **Cost** per request, per user, per feature | Who is burning the budget?           |
| **Time-to-first-token (TTFT)** | Does the user see anything fast?              |
| **End-to-end latency**      | How long is the full agent loop?                 |
| **Tool-call latency** (per tool) | Which tool is the bottleneck?               |
| **LLM call count** per request | Is the agent looping more than expected?      |
| **Tool call count** per request | Same — agent over-tooling?                   |
| **Retrieval recall@K**      | Are we missing chunks before the LLM even sees them? |
| **Error rate** by stage     | Where in the pipeline does it break?             |
| **Cache hit rate** (if applicable) | Are we paying for the same prompt twice?  |

Track all of these per request, then aggregate to dashboards. Without per-request granularity, you can't debug specific failures.

---

## A Structured-Log Schema for LLM Requests

If you take nothing else from these notes, take this. Log this object at the end of every agent request:

```python
import json, time, uuid

def log_request(record: dict):
    print(json.dumps(record))   # CloudWatch / OTel logs picks this up

def handler(user_msg, user_id):
    req_id = str(uuid.uuid4())
    t0 = time.monotonic()
    record = {
        "ts":               time.time(),
        "request_id":       req_id,
        "user_id":          user_id,
        "feature":          "candidate-summary",   # what part of the product
        "input_tokens":     0,
        "output_tokens":    0,
        "model_calls":      [],     # [{model, in_tokens, out_tokens, latency_ms}]
        "tool_calls":       [],     # [{name, latency_ms, success}]
        "retrieval":        None,   # {top_k, recall_score} when relevant
        "ttft_ms":          None,
        "total_ms":         None,
        "cost_usd":         0.0,
        "status":           "ok",
        "error":            None,
    }
    try:
        # ... agent loop, populating record as you go ...
        return result
    except Exception as e:
        record["status"] = "error"
        record["error"]  = str(e)
        raise
    finally:
        record["total_ms"] = int((time.monotonic() - t0) * 1000)
        log_request(record)
```

One JSON line per request. CloudWatch Logs Insights, Athena, or any log-processing tool can slice this every which way. You can answer:

- "What did we spend on `candidate-summary` last week per user?"
- "Which feature has the slowest p95 LLM call?"
- "Why is request `abc-123` taking 12 seconds?"

> Optimize for one log line per logical request, not per LLM call. Aggregate sub-call detail into arrays inside the record. Your future self trying to debug at 2 AM will thank you.

---

## Computing Cost Per Request

Most providers publish per-token prices that vary by model. Hard-code them and compute on the fly:

```python
PRICES_USD_PER_1K = {
    # input, output (per 1000 tokens)
    "gpt-4o":              (0.0025, 0.0100),
    "gpt-4o-mini":         (0.00015, 0.00060),
    "claude-3-5-sonnet":   (0.003,  0.015),
    "claude-3-5-haiku":    (0.0008, 0.004),
    "amazon.titan-text-express-v1": (0.0008, 0.0016),
}

def cost_for(model: str, in_tok: int, out_tok: int) -> float:
    pin, pout = PRICES_USD_PER_1K[model]
    return (in_tok / 1000) * pin + (out_tok / 1000) * pout
```

Add the result into the request record. Now you have dollars per user, per feature, per day — just by aggregating.

> Keep the price table in code (or a config file) reviewed alongside model upgrades. Stale price data quietly distorts every cost dashboard you build.

---

## TTFT vs End-to-End Latency

For chat-like UX, **time-to-first-token** matters more than total time. Users tolerate a long answer if the first words appear fast.

```python
import time

t_request = time.monotonic()
ttft_ms = None

stream = client.chat.completions.create(model="gpt-4o", messages=[...], stream=True)
for chunk in stream:
    if ttft_ms is None:
        ttft_ms = int((time.monotonic() - t_request) * 1000)
    # ... append delta to response ...
total_ms = int((time.monotonic() - t_request) * 1000)
```

Track p50 / p95 / p99 separately for TTFT and total. Streaming hides total latency from the user; non-streaming makes them stare at a spinner.

> If TTFT is slow and total is fast → cold model / API issue. If TTFT is fast and total is slow → output too long. The two metrics tell different stories.

---

## Alerting Thresholds That Don't Cry Wolf

Static thresholds break the moment you change a model or feature. Better: alert on **rate-of-change** and **percentile breaches**:

| Alert                                | Threshold                                     |
|--------------------------------------|-----------------------------------------------|
| Cost / hour up vs prior hour         | +200% — almost always a runaway loop          |
| p95 LLM latency                      | >2× rolling 24h average                       |
| Error rate per feature                | >5% over 15 min (low-traffic features need higher window) |
| Token budget per user / day          | Per-user hard cap — circuit-break on hit      |
| Single request > N tokens             | Often a sign of context leaking unbounded     |
| `retrieval recall@K = 0` rate        | >10% over 30 min — your index is stale        |

Tie these to a one-line action — the page should tell you what to look at first, not just that something is broken.

---

## Budgets and Circuit Breakers

If a user / feature hits a token cap, **fail fast** instead of running the full agent loop:

```python
DAILY_TOKEN_BUDGET = {"free": 50_000, "pro": 1_000_000}

def check_budget(user_id, tier):
    used = redis.get(f"tokens:{user_id}:{today()}") or 0
    if int(used) >= DAILY_TOKEN_BUDGET[tier]:
        raise BudgetExceeded(f"User {user_id} hit daily token cap")

def handler(event):
    check_budget(event["user_id"], event["tier"])
    # ... run agent ...
    redis.incrby(f"tokens:{event['user_id']}:{today()}", total_tokens_used)
```

Rate-limit before model invocation, not after — once you've made the API call, you've already paid for the tokens.

> The cheapest LLM call is the one you don't make. Caching, budgets, and shortcuts (rule-based pre-checks) are infrastructure, not optimisations. Build them in early.

---

## Common Mistakes

| Mistake                                          | Symptom                                          |
|--------------------------------------------------|--------------------------------------------------|
| Tracing only the top-level handler               | Can't see which sub-call is slow                 |
| Logging prompt + completion as a single string   | Can't analyse tokens / costs separately          |
| Forgetting to record the model id served         | Provider auto-upgraded, you don't notice         |
| Charging cost in product currency without taxes  | Reports look low; surprise bill at month end     |
| Using static latency alerts                      | Wakes you at 3 AM whenever traffic spikes        |
| Sending raw PII into LangSmith / OTel attributes | Compliance violation; redact before exporting    |
| Not keeping a real eval set                      | Can't tell whether a "fix" actually fixed anything |
| Counting only LLM cost, ignoring retrieval/embedding | Total bill is 40% larger than you think      |

---

# Quick Reference — All Four Concepts

```
WHY AGENT OBSERVABILITY IS DIFFERENT
  Web app trace ends at HTTP latency; agent trace must capture:
    tokens, cost, tool calls, retrieval, prompts, model id, decisions
  Traces are TREES — every LLM call can spawn tools / sub-LLMs
  You can't fix what you can't see → instrument before you ship

LANGSMITH — THE ZERO-CONFIG OPTION
  pip install langsmith  +  3 env vars (LANGCHAIN_TRACING_V2, _API_KEY, _PROJECT)
  All LangChain runs traced automatically
  @traceable(run_type=...) for non-LangChain code
  Run types: chain / llm / tool / retriever / parser
  Datasets + evals built in — save real failures, re-run on every change
  Best when you're on LangChain and team < 50 engineers

OPENTELEMETRY — THE VENDOR-NEUTRAL OPTION
  TRACE → SPAN → ATTRS / EVENTS
  GenAI semantic conventions: gen_ai.system / gen_ai.request.model /
                               gen_ai.usage.input_tokens / gen_ai.tool.name
  Auto-instrumentation: opentelemetry-instrumentation-openai / -langchain
  Manual spans for custom tools — wrap with tracer.start_as_current_span
  Exports to X-Ray (via ADOT collector or Lambda layer), Datadog, Grafana, etc.

LANGSMITH + OTEL TOGETHER
  LangSmith for ML eng (datasets, prompt diff, evals)
  OTel for SRE (latency, alerting, AWS-stitched traces)
  Both run from the same code without conflict

METRICS THAT MATTER
  tokens in/out per request
  cost USD per request / user / feature
  TTFT (streaming UX)
  end-to-end latency p50/p95/p99
  tool-call latency per tool
  LLM call count per request (loop detection)
  retrieval recall@K (when applicable)
  error rate per stage
  cache hit rate

ONE-LINE-PER-REQUEST LOG SCHEMA
  request_id, user_id, feature
  input_tokens, output_tokens, cost_usd
  model_calls[]  — {model, in_tok, out_tok, latency_ms}
  tool_calls[]   — {name, latency_ms, success}
  retrieval      — {top_k, recall_score}
  ttft_ms, total_ms, status, error

COST COMPUTATION
  Hard-code per-1K-token prices, look up by model id
  cost = in_tok/1000 * price_in + out_tok/1000 * price_out
  Review the price table on every model upgrade — stale data = wrong dashboards

ALERTING THAT DOESN'T CRY WOLF
  cost/hour: +200% vs prior hour (runaway loops)
  p95 latency: >2× rolling 24h
  error rate: >5% / 15min per feature
  per-user token budget: hard cap with circuit break
  retrieval recall@K = 0 rate: >10% / 30min (stale index)

BUDGETS / CIRCUIT BREAKERS
  Cheapest LLM call = the one you don't make
  Check budget BEFORE invoking model — once called, you've paid
  Build budgets / caching / rule-based shortcuts as infra, not optimisations

COMMON MISTAKES
  Tracing only the top-level handler
  Logging prompt + completion as one string (no token analysis)
  Forgetting to record served-model id (silent provider upgrade)
  Static latency alerts (always cry wolf on traffic spikes)
  Sending PII to LangSmith / OTel attributes (compliance issue)
  Counting only LLM cost — missing retrieval / embedding spend
  No saved eval set — can't tell if a fix actually fixed it
```
