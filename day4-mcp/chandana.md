# Day 4 — Chandana (AWS)

Notes, labs, and experiments for Day 4.

## Topics
- [x] Hosting MCP servers on AWS — Lambda for short tools vs Fargate for long-running ones
- [x] Securing MCP — auth, scopes, rate-limiting, handling secrets the server needs

---

## 1. Hosting MCP servers on AWS

### Rule of thumb
| | Lambda | Fargate |
|---|---|---|
| Connection | short, request-response | persistent SSE stream |
| State | stateless | stateful |
| Max duration | 15 min | unlimited |
| Cost (example) | ~$1.20/mo per 1M calls | ~$14/mo always-on |
| Best for | bursty / unpredictable | steady / long-running |

---

### Lambda — use when
- Tool completes fast and returns a result (search, SQL query, API call, send message)
- No connection needs to stay open between calls
- Nothing needs to be remembered between invocations

**Key setup decisions**
- Use **Lambda Function URL**, not API Gateway (API GW has a 29s timeout that kills SSE)
- Offload any state to **DynamoDB** or **S3**
- Coldstart hurts latency → add **Provisioned Concurrency** if p99 matters

---

### Fargate — use when
- Client holds a **persistent SSE connection** (MCP's default transport)
- Tool needs in-process memory across multiple calls (browser session, agent loop)
- Work runs longer than 15 min, or watches/polls for events (tail logs, CI monitor)

**Key setup decisions**
- ECS task behind an **ALB with sticky sessions** — critical, without this SSE breaks on re-routes
- Container image lives in **ECR** — same pattern as Rooman apps
- Scale on CPU % via **ECS Auto Scaling**

---

### Hybrid pattern (production default)

```
Fargate MCP router  ← always-on, holds the SSE connection
  ├── Lambda        ← dispatches fast discrete tools (< 1s)
  └── Fargate task  ← spawns for long-running / stateful work
```

---

## 2. Securing MCP

Every MCP server needs four layers. They run in order — a request is rejected at the first layer it fails.

```
Caller (Bearer token)
  ↓
[1] API Gateway / ALB    — rate limit before traffic hits your code
  ↓
[2] Auth middleware      — is this token valid?           → 401 if not
  ↓
[3] Scope middleware     — can this token call this tool? → 403 if not
  ↓
[4] Rate limiter         — has this caller exceeded quota? → 429 if yes
  ↓
Tool handler             — runs with secrets fetched at runtime
  ↓
Downstream (DB, APIs, AWS services)
```

---

### Layer 1 — Auth (who are you?)

**API key** — use this for internal tools (Rooman ERP/CRM)
- Client sends `Authorization: Bearer <key>`
- Middleware validates before any tool logic runs
- Key stored in **Secrets Manager**, injected as env var — never hardcoded

**OAuth 2.0** — use this only if external users need delegated access
- Client gets a short-lived JWT from an IdP (Cognito, Google)
- Server verifies JWT signature via provider's JWKS endpoint

> Rooman = internal → API key is enough. OAuth only when external users appear.

```python
import jwt   # pip install pyjwt[cryptography]

def validate_jwt(token: str, jwks_url: str, audience: str) -> dict:
    jwks_client = jwt.PyJWKClient(jwks_url)
    signing_key = jwks_client.get_signing_key_from_jwt(token)
    return jwt.decode(token, signing_key.key, algorithms=["RS256"], audience=audience)
    # auto-checks: signature + expiry (exp) + audience (aud)
    # raises jwt.ExpiredSignatureError / jwt.InvalidTokenError on failure
```

---

### Layer 2 — Scopes (what can you do?)

- Each tool maps to required scopes
- Token must carry all required scopes, or call is rejected with **403**
- **Deny by default** — tools with no scope definition should reject everyone

```python
TOOL_SCOPES = {
    "search_crm":    ["crm:read"],
    "update_lead":   ["crm:write"],
    "delete_record": ["crm:admin"],   # most destructive = narrowest scope
}

def check_scope(token_scopes: list, tool_name: str):
    required = TOOL_SCOPES.get(tool_name, [])
    if not all(s in token_scopes for s in required):
        raise PermissionError(f"Missing scope: {required}")
```

---

### Layer 3 — Rate limiting (how often?)

Algorithm: **token bucket** — each caller gets N tokens per window, refills over time

| Tool type | Limit |
|---|---|
| Normal (search, read) | 60 calls / min |
| Expensive (compute, browser) | 10 calls / min |

**Where to store the counter**
- Lambda → **DynamoDB** (no in-process state between invocations)
- Fargate → in-process (`slowapi`) or **ElastiCache Redis**
- API Gateway Usage Plans → first line of defence, runs before your code

---

### Layer 4 — Secrets (what credentials does the server use?)

**Never**
- Hardcode in source code
- Bake into Docker image (`docker build --build-arg` leaks into image layers)
- Store sensitive values in plain SSM Parameter Store

**Always** — fetch from **AWS Secrets Manager** at runtime via IAM role

```python
from functools import lru_cache
import boto3, json

@lru_cache(maxsize=None)   # fetches once per container lifecycle, reuses on warm calls
def get_secret(name: str) -> dict:
    client = boto3.client("secretsmanager", region_name="us-west-2")
    return json.loads(client.get_secret_value(SecretId=name)["SecretString"])

# usage inside tool handler:
secrets = get_secret("rooman/mcp/prod")
db_password = secrets["db_password"]
```

**Fargate shortcut** — ECS injects secrets at container start, no SDK call needed

```json
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:us-west-2:123456789:secret:rooman/mcp/prod:db_password::"
  }
]
```

Code just reads `os.environ["DB_PASSWORD"]` — ECS handles the Secrets Manager fetch.

IAM role must have `secretsmanager:GetSecretValue` on specific ARNs only — nothing broader.