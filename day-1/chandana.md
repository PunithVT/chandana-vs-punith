# Day 1 — Chandana (AWS)

Notes, labs, and experiments for Day 1.

Picking up from the [Day 1 topics](./README.md) — focusing on the AWS-heavy ones:

- [ ] What is AWS Lambda — serverless compute model, cold starts, execution environments
- [ ] Deploying a Python Lambda — packaging dependencies, layers, console vs CLI
- [ ] Triggering Lambda — API Gateway, S3 events, EventBridge, SQS

## Notes

# AWS Lambda — Complete Notes

---

# CONCEPT 1 — What is Lambda & How It Works

---

## Serverless vs EC2

**EC2** = renting a server. Runs 24/7. You manage OS, Python, scaling, patching. Pay even at zero traffic.

**Lambda** = run a function when triggered. No server to manage. Pay only while it runs. Zero traffic = zero cost.

```
EC2:     You → rent server → install everything → pay 24/7
Lambda:  You → write function → AWS runs it when triggered → pay per ms
```

| Thing       | EC2                       | Lambda                    |
|-------------|---------------------------|---------------------------|
| You manage  | OS, runtime, patching     | Nothing — just code       |
| Billing     | Per hour (always on)      | Per 100ms (when running)  |
| Idle cost   | Full price                | Zero                      |
| Scaling     | Manual                    | Automatic                 |
| Max runtime | Unlimited                 | 15 minutes                |

> Rule: Always on → EC2. Only runs when triggered → Lambda.

---

## Pricing

**Requests:** First 1M/month free → $0.20 per 1M after.

**Duration:** Charged per GB-second of memory used.
- Free tier: 400,000 GB-seconds/month
- After: $0.0000166667 per GB-second

```
Example: 512MB memory, 200ms runtime, 5M calls/month
  Requests = (5M - 1M free) × $0.0000002     = $0.80
  Duration = 5M × 0.2s × 0.5GB × $0.0000167 = $0.83
  Total    = ~$1.63/month   (vs EC2 t3.micro = ~$9/month idle)
```

---

## Cold Start vs Warm Start

Lambda containers are killed after ~15 min idle. Next request pays startup cost = **cold start**.

```
COLD START (new container):
  [1] Download code         50–300ms   ← AWS
  [2] Boot Firecracker VM   50–200ms   ← AWS (+500ms if VPC)
  [3] Start Python          50–150ms   ← AWS
  [4] Run your imports      100ms–2s   ← YOU control this
  [5] Run handler()         your logic

WARM START (container reused):
  [5] Run handler() only    ~1ms overhead
```

**Reduce cold starts:**

| Action                          | Impact  |
|---------------------------------|---------|
| Minimize imports / package size | High    |
| Avoid VPC unless needed         | High    |
| Move heavy imports inside fn    | High    |
| Use Python 3.12                 | Medium  |
| Increase memory (more CPU)      | Medium  |
| Provisioned Concurrency         | Eliminates (paid) |

```python
# Bad — pandas loads on every cold start
import pandas as pd

# Good — only loads when that path runs
def handler(event, context):
    if event.get("run_report"):
        import pandas as pd
```

---

## Firecracker MicroVM

Every Lambda runs inside a **Firecracker microVM** — AWS-built, open-source (2018).
Not Docker (shares kernel = less secure). Not full VM (too slow to start). Middle path.

```
Physical server
└── Firecracker hypervisor
    └── MicroVM (your sandbox)
        ├── /var/task/    ← your code
        ├── /tmp/         ← temp storage (512MB–10GB)
        ├── Python 3.12
        └── Your handler
```

Boots in **~125ms** (vs 1–2s for full VMs). Hardware-level isolation via Intel VT-x / AMD-V.

**Lifecycle:**
```
INIT     → VM boots, Python starts, imports run       (cold start only)
INVOKE   → handler() called, response returned        (every request)
FROZEN   → VM paused, memory preserved                (between requests)
SHUTDOWN → VM killed after ~15min idle                (next = cold start)
```

| Fact                                | Implication                                   |
|-------------------------------------|-----------------------------------------------|
| Each concurrent request = new VM    | Globals NOT shared across concurrent requests |
| FROZEN preserves memory             | DB connections reused on warm starts          |
| /tmp persists in FROZEN             | Can cache files across warm invocations       |
| SHUTDOWN wipes everything           | Always check /tmp before assuming file exists |

---

# CONCEPT 2 — Deploying a Python Lambda

---

## Packaging Dependencies

Lambda ships with only stdlib + boto3. Everything else must be **bundled inside your ZIP**.

**Correct ZIP structure** — files at root, not inside a subfolder:
```
function.zip
├── lambda_function.py   ← your code
├── requests/            ← bundled dep
└── urllib3/
```

**Build script:**
```bash
pip install -r requirements.txt -t ./package/
cp lambda_function.py ./package/
cd package && zip -r ../function.zip . && cd ..
aws lambda update-function-code --function-name my-fn --zip-file fileb://function.zip
```

> `cd package` before zipping is critical — zip from outside = subfolder = handler not found.

**Size limits:**

| Method            | Limit   |
|-------------------|---------|
| ZIP direct upload | 50 MB   |
| ZIP via S3        | 250 MB  |
| Container image   | 10 GB   |

---

## Platform Compatibility (C Extensions)

Packages with C extensions compiled on Mac/Windows **fail on Lambda** (Amazon Linux x86_64).
Common culprits: `psycopg2`, `numpy`, `pandas`, `Pillow`, `cryptography`, `lxml`.

**Fix — build inside Lambda's Docker image:**
```bash
docker run --rm -v $(pwd):/var/task \
  public.ecr.aws/lambda/python:3.12 \
  pip install -r requirements.txt -t /var/task/package/
```

**psycopg2 — 3 options:**

| Option              | When                            |
|---------------------|---------------------------------|
| `psycopg2-binary`   | Most cases — simplest           |
| Community Layer     | Already using Layers            |
| `pg8000`            | Pure Python, zero C extensions  |

---

## Lambda Layers

A Layer is a separate ZIP mounted at `/opt/` — shared across multiple functions.

**Required structure inside Layer ZIP:**
```
my-layer.zip
└── python/lib/python3.12/site-packages/
    └── requests/
```

```bash
mkdir -p layer/python/lib/python3.12/site-packages
pip install requests -t layer/python/lib/python3.12/site-packages/
cd layer && zip -r ../my-layer.zip python/ && cd ..

aws lambda publish-layer-version --layer-name my-deps \
  --zip-file fileb://my-layer.zip --compatible-runtimes python3.12

aws lambda update-function-configuration \
  --function-name my-fn \
  --layers arn:aws:lambda:us-west-2:123:layer:my-deps:1
```

Limits: max 5 layers per function, 250 MB total unzipped (code + all layers).

| Use case                      | Method          |
|-------------------------------|-----------------|
| Single fn, small deps         | Bundle in ZIP   |
| Multiple fns share same deps  | Layer           |
| numpy / ML / >250 MB          | Container image |

---

## Console vs CLI

**Console** — explore, view config, test with custom events, read logs. Cannot be scripted.

**CLI** — deploy, automate, CI/CD. Every action is reproducible.

> Rule: Console to understand. CLI to do.

```bash
# Create
aws lambda create-function --function-name my-fn \
  --runtime python3.12 --handler lambda_function.handler \
  --role arn:aws:iam::123:role/exec-role --zip-file fileb://function.zip

# Deploy code only (no config change)
aws lambda update-function-code --function-name my-fn --zip-file fileb://function.zip

# Update config only (no redeploy)
aws lambda update-function-configuration --function-name my-fn --timeout 60 --memory-size 512

# Invoke and see output
aws lambda invoke --function-name my-fn \
  --payload '{"action":"test"}' --cli-binary-format raw-in-base64-out out.json

# Stream live logs
aws logs tail /aws/lambda/my-fn --follow --format short
```

> `--cli-binary-format raw-in-base64-out` required in CLI v2 for JSON payloads.

---

## Container Image Deployment

Use when deps exceed 250 MB or you need to test locally with full fidelity.

```dockerfile
FROM public.ecr.aws/lambda/python:3.12
COPY requirements.txt .
RUN pip install -r requirements.txt --no-cache-dir
COPY lambda_function.py .
CMD ["lambda_function.handler"]
```

```bash
docker build --platform linux/amd64 -t my-lambda .
docker tag my-lambda:latest 123.dkr.ecr.us-west-2.amazonaws.com/my-lambda:latest
docker push 123.dkr.ecr.us-west-2.amazonaws.com/my-lambda:latest
aws lambda update-function-code --function-name my-fn \
  --image-uri 123.dkr.ecr.us-west-2.amazonaws.com/my-lambda:latest

# Test locally before deploying
docker run --rm -p 9000:8080 my-lambda:latest
curl -X POST http://localhost:9000/2015-03-31/functions/function/invocations -d '{"test":1}'
```

> Always `--platform linux/amd64` on Apple Silicon — without it ARM image = Lambda fails.

---

# CONCEPT 3 — Triggering Lambda

---

## Three Invocation Modes

| Mode           | Caller waits? | Auto-retry? | Examples                      |
|----------------|---------------|-------------|-------------------------------|
| Synchronous    | Yes           | No          | API Gateway, direct invoke    |
| Asynchronous   | No            | Yes (2×)    | S3, SNS, EventBridge          |
| Poll-based     | No            | Batch back  | SQS, Kinesis, DynamoDB Streams|

---

## Trigger 1 — API Gateway (Synchronous)

Client → HTTP request → API Gateway → Lambda → waits → HTTP response back.

**Key event fields:**
```python
event["httpMethod"]                         # "GET", "POST"
event["path"]                               # "/users"
event.get("queryStringParameters") or {}   # None when no params — use `or {}`
json.loads(event["body"])                   # body is a STRING
```

**Response format — must match exactly:**
```python
def handler(event, context):
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"ok": True})    # body must be a STRING
    }
```

> Hard limit: API Gateway times out at **29 seconds**. Set Lambda timeout ≤ 28s.
> Both request `body` and response `body` are strings — always `json.loads` / `json.dumps`.

---

## Trigger 2 — S3 Events (Asynchronous)

File uploaded/deleted → S3 notifies Lambda async → Lambda processes. No response to S3.

```python
from urllib.parse import unquote_plus
import boto3

s3 = boto3.client("s3")   # global — reused on warm starts

def handler(event, context):
    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key    = unquote_plus(record["s3"]["object"]["key"])  # always decode
        obj    = s3.get_object(Bucket=bucket, Key=key)
        data   = obj["Body"].read()
        # process...
        s3.put_object(Bucket="output-bucket", Key=f"processed/{key}", Body=data)
```

> S3 key is **URL-encoded** — `my file.jpg` arrives as `my+file.jpg`. Always `unquote_plus()`.
> Write output to a **different bucket or prefix** — same bucket = infinite loop.
> On failure Lambda retries **2 more times**. Configure a DLQ to catch persistent failures.

---

## Trigger 3 — EventBridge (Asynchronous)

Two uses: **scheduled cron** and **event-driven routing**.

**Scheduled (replaces EC2 cron):**
```bash
aws events put-rule --name daily-job --schedule-expression "cron(0 2 * * ? *)"
aws events put-targets --rule daily-job \
  --targets "Id=1,Arn=arn:aws:lambda:us-west-2:123:function:my-fn"
```

```
cron(0 2 * * ? *)        → daily 2am UTC
cron(0 9 ? * MON-FRI *)  → weekdays 9am
rate(5 minutes)           → every 5 minutes
```

> Use `?` for day-of-month OR day-of-week — never `*` in both.

**Event-driven (custom events between services):**
```python
# Service A publishes
boto3.client("events").put_events(Entries=[{
    "Source": "myapp.orders",
    "DetailType": "OrderPlaced",
    "Detail": json.dumps({"order_id": 42})
}])

# Lambda receives — payload is in event["detail"]
def handler(event, context):
    order_id = event["detail"]["order_id"]
```

---

## Trigger 4 — SQS (Poll-based)

Producer → SQS queue ← Lambda polls → processes batch. Lambda manages polling via **Event Source Mapping**.

**Critical — partial batch response (must implement this):**
```python
def handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            body = json.loads(record["body"])   # body is a STRING
            process(body)
        except Exception:
            failures.append(record["messageId"])

    # Only failed messages stay in queue — successes auto-deleted
    return {"batchItemFailures": [{"itemIdentifier": mid} for mid in failures]}
```

**Setup:**
```bash
aws lambda create-event-source-mapping \
  --function-name my-fn \
  --event-source-arn arn:aws:sqs:us-west-2:123:my-queue \
  --batch-size 10 \
  --function-response-types ReportBatchItemFailures
```

> Without `ReportBatchItemFailures` — one failure = entire batch retried = duplicate processing.
> Set `maxReceiveCount=3` on queue + configure a **DLQ** for poison messages.
> Standard queue = at-least-once delivery. FIFO = exactly-once, strict order, lower throughput.

---

# Quick Reference — All Three Concepts

```
LIMITS
  Max runtime         → 15 min
  Memory              → 128 MB – 10 GB
  /tmp storage        → 512 MB – 10 GB
  Concurrent default  → 1,000/region
  ZIP size            → 50 MB direct / 250 MB via S3
  Max layers          → 5 per function / 250 MB total
  Container image     → 10 GB

COLD START ORDER
  Download → Firecracker VM → Python → imports → handler()
  Warm start = handler() only (~1ms overhead)

FIRECRACKER
  Purpose-built microVM, boots ~125ms, hardware isolation, open-source (AWS 2018)
  Concurrent requests = separate VMs = globals NOT shared across concurrent calls

COST
  1M requests/month free → $0.20/1M after
  400K GB-seconds/month free → $0.0000166667/GB-second after

PACKAGING
  ZIP files must be at root — never in a subfolder
  C extensions (psycopg2, numpy) → build inside Lambda Docker image
  psycopg2 → psycopg2-binary or pg8000
  Layer path must be → python/lib/python3.12/site-packages/
  Apple Silicon → always --platform linux/amd64

CLI ESSENTIALS
  update-function-code          → redeploy code (no config change)
  update-function-configuration → change timeout/memory/env (no redeploy)
  invoke --cli-binary-format raw-in-base64-out → test with JSON
  logs tail /aws/lambda/{name} --follow → live logs

TRIGGER MODES
  Synchronous  (API Gateway)  → caller waits, no auto-retry
  Asynchronous (S3, EventBridge) → fire and forget, retries 2×, use DLQ
  Poll-based   (SQS)          → Lambda polls, batch processing

TRIGGER GOTCHAS
  API Gateway  → body always string (json.loads/dumps), 29s hard timeout
  S3           → key is URL-encoded (unquote_plus), avoid same-bucket writes
  EventBridge  → cron needs ? not *, payload is in event["detail"]
  SQS          → must enable ReportBatchItemFailures or get duplicates on retry
```
