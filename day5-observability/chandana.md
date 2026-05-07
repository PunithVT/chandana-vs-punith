# Day 5 — Chandana (AWS)

Notes, labs, and experiments for Day 5.

## Topics
- [x] CloudWatch Logs + Logs Insights — structured logging and useful queries
- [x] CloudWatch Metrics + Alarms — custom metrics, dashboards, what's worth alerting on
- [x] AWS X-Ray — distributed tracing, service map, sampling

---

## The big picture — three tools, three questions

| Tool | Question it answers |
|---|---|
| CloudWatch Logs | What exactly happened? What were the values? |
| CloudWatch Metrics | How many? How fast? How often? |
| X-Ray | Where did the time go? Which service is the bottleneck? |

Use metrics to **detect** the problem → logs to **understand** the context → traces to **find** the cause.

---

## 1. CloudWatch Logs + Logs Insights

### What it is

Every AWS service (Lambda, ECS, EC2, API Gateway) automatically ships its output to CloudWatch Logs.

Logs are organised in two levels:
- **Log group** — one per service (e.g. `/aws/lambda/rooman-api`)
- **Log stream** — one per container/instance/invocation inside that group

You don't search streams manually. You query across the entire log group using **Logs Insights**.

---

### Structured logging — the right way to log

If you log plain strings, you can only do text search.
If you log **JSON**, Logs Insights can filter on fields, aggregate values, and compute statistics — like SQL on your logs.

Plain string log → hard to query:
```
2024-01-15 INFO User 42 placed order 99 for $150.00 in 230ms
```

JSON log → every field is queryable:
```json
{"event": "order_placed", "user_id": 42, "order_id": 99, "amount": 150.00, "duration_ms": 230}
```

**Rule: always log JSON in production.**

---

### Logs Insights — query language

Runs over a time range you pick. Returns results in seconds even across millions of lines.

Basic structure:
```
fields @timestamp, @message
| filter <condition>
| stats <aggregation> by <field>
| sort <field> desc
| limit <N>
```

Useful queries:
- All errors → `filter level = "ERROR"`
- Slow requests → `filter duration_ms > 1000`
- Error rate over time → `stats count(*) as errors by bin(5m)`
- P95/P99 latency → `stats percentile(duration_ms, 95) as p95 by endpoint`

---

### Lambda auto-injected fields (free, no code needed)

Lambda writes these fields into every log line automatically:

| Field | What it tells you |
|---|---|
| `@duration` | Total invocation time (ms) |
| `@maxMemoryUsed` | Peak memory — alert if approaching configured limit |
| `@initDuration` | Cold start time — only present on cold starts |
| `@requestId` | Unique per invocation — use to correlate logs across services |
| `@billedDuration` | What you're charged for |

---

### Log retention — set this immediately

By default CloudWatch keeps logs **forever** and charges per GB stored.
Set a retention policy on every log group — 30 days is a good default.
Production audit logs: 90 days.

---

## 2. CloudWatch Metrics + Alarms

### What metrics are

A metric is a **time series of numbers** — a data point every N seconds/minutes.
Every AWS service publishes metrics automatically. You can also publish your own.

Metrics have **dimensions** — key/value pairs to slice data.
Example: `Duration` metric on Lambda has a `FunctionName` dimension → one time series per function.

---

### Key metrics to watch per service

**Lambda**

| Metric | Watch for |
|---|---|
| `Errors` | Any non-zero value in production |
| `Duration` (p99) | Approaching the function timeout |
| `Throttles` | Concurrency limit hit — users getting rejected |
| `ConcurrentExecutions` | Sudden spike or unexpected drop |

**ECS / Fargate**

| Metric | Watch for |
|---|---|
| `CPUUtilization` | > 80% sustained → scale out |
| `MemoryUtilization` | > 85% → risk of container being killed |
| `RunningTaskCount` | Sudden drop → tasks crashing |

**RDS**

| Metric | Watch for |
|---|---|
| `DatabaseConnections` | Near `max_connections` → connection pool exhausted |
| `FreeStorageSpace` | < 20% → alert before it fills up |
| `ReadLatency` / `WriteLatency` | Spike → slow queries or I/O saturation |

---

### Custom metrics — your own business numbers

When AWS metrics aren't enough, publish your own. Examples:
- Orders placed per minute
- Embedding generation duration
- Background job failures

Custom metrics cost $0.30/metric/month — negligible. Use a clear namespace like `MyApp/Production` to keep them organised in the console.
---

### Alarms — what's worth alerting on

An alarm watches one metric and triggers when it crosses a threshold.
Three states: `OK` → `ALARM` → `INSUFFICIENT_DATA`

**Alert on these:**
- Lambda `Errors` > 0 in production
- RDS `FreeStorageSpace` < 20%
- ECS `RunningTaskCount` drops below desired
- API Gateway `5XXError` rate > 1%
- Lambda `Throttles` > 0

**Don't alert on these:**
- CPU spikes shorter than 5 minutes
- Dev environment errors
- P50 latency
- Memory usage below 70%

> Alert fatigue is real — an alarm that fires every day gets ignored.
> Start with 5 high-signal alarms, not 30 noisy ones.

---

### Dashboards

Group related metrics into one view. Build one dashboard per service, not one giant account dashboard.

A good Rooman dashboard has: Lambda errors, Lambda p99 latency, ECS running task count, RDS connections, RDS free storage — all on one screen.

---

## 3. AWS X-Ray — Distributed Tracing

### What problem it solves

Logs tell you what happened on one service.
Metrics tell you how much and how often.
Neither tells you **where the time went across multiple services**.

Example — a user request takes 2.3 seconds:
```
API Gateway      →   5ms
Lambda cold start → 400ms   ← cold start
Lambda handler   →  50ms
RDS query        → 1800ms   ← the real problem
Response         →  45ms
```

X-Ray gives you this breakdown for every request.

---

### Core concepts

**Trace** — the full journey of one request across all services, end to end.

**Segment** — one service's contribution to the trace (e.g. Lambda ran for 500ms total).

**Subsegment** — a unit of work inside a segment (e.g. one RDS query took 1800ms, one S3 put took 12ms).

**Service map** — a visual graph auto-generated from trace data showing all services, how they connect, latency on each edge, and error rates. Start here when something is slow.

**Sampling** — X-Ray does not trace 100% of requests (too expensive). It samples a percentage. Default: 5% of requests + always the first request each second. You can raise sampling on critical paths (e.g. 50% on `/api/orders`).

**Annotations** — key/value pairs attached to a trace that are **indexed and searchable** (e.g. `order_id = "99"`).

**Metadata** — extra detail attached to a trace that is **not indexed** — use for large debug payloads like full request bodies.

---

### How X-Ray works on each service

**Lambda** — enable Active Tracing in the function config. Add the X-Ray SDK to your code. It auto-patches boto3, psycopg2, and requests — every AWS SDK call and DB query becomes a subsegment automatically. No manual instrumentation needed for basic tracing.

**ECS / Fargate** — runs an X-Ray daemon as a **sidecar container** in the same ECS task. Your app sends trace data to it via UDP on `localhost:2000`. The daemon batches and forwards to X-Ray. Add it to your task definition alongside your app container. It uses ~32 CPU units and ~256MB memory — lightweight.

IAM role needs: `xray:PutTraceSegments` and `xray:PutTelemetryRecords`.

---

### What to look at in the X-Ray console

**Service map** — start here. Visual graph of all services. Click any edge to see latency distribution and error rate between those two services. Immediately shows which service is the bottleneck.

**Traces list** — searchable list of individual requests. Filter by annotation (`order_id = "99"`), duration (`> 2 seconds`), or error status. Find the specific slow or broken request.

**Trace detail** — timeline view of one request showing every segment and subsegment with exact milliseconds. This is where you see the slow RDS query, the cold start, or the downstream API timeout.

---

### Sampling — control what gets traced and what it costs

Default (5% + 1/sec) is fine for low traffic.
For high-traffic production: keep 5% on general traffic, raise to 50% on critical paths like checkout or payment.
Setting sampling rules per URL path and HTTP method lets you be precise — trace everything that matters, sample the rest.

---

## How all three work together in practice

```
Something is wrong — users report slowness
         ↓
CloudWatch Metrics    p99 latency spiked to 3s at 14:32
         ↓
X-Ray Service Map     RDS segment is red — high latency on DB edge
         ↓
X-Ray Trace Detail    One query taking 2.8s — missing index on orders table
         ↓
CloudWatch Logs       Confirm with structured logs — which user, which order, full context
         ↓
Fix deployed, metrics return to normal, alarm clears
```

Metrics → detect  
X-Ray → locate  
Logs → confirm and understand