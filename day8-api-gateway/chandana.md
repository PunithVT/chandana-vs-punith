# Day 8 — Chandana (AWS)

Notes, labs, and experiments for Day 8.

Picking up from the [Day 8 topics](./README.md) — focusing on the AWS-heavy ones:

- [ ] API Gateway — REST vs HTTP API, when to pick which, throttling and usage plans
- [ ] Custom domains, mTLS, and Cognito / JWT authorizers
- [ ] Streaming agent responses — SSE vs WebSockets, the 29s API GW timeout gotcha

## Notes

_to be filled in as I go_

# Day 8 — API Gateway

Notes covering REST vs HTTP API, custom domains, mTLS, Cognito/JWT authorizers, and streaming agent responses.

---

## What is API Gateway?

API Gateway is AWS's managed front door for your backends. Instead of exposing your Lambda, EC2, or ECS service directly to the internet, you put API Gateway in front. It handles the HTTP layer — routing requests to the right backend, enforcing auth, rate limiting, and logging — and gives you a stable public URL to hand to clients.

The mental model: a client calls `https://api.example.com/orders`, API Gateway receives it, checks auth, forwards it to the right Lambda or service, gets the response, and sends it back. The backend is never exposed directly.

---

## Topic 1 — REST API vs HTTP API

This is the most common confusion with API Gateway because the naming is misleading. Both support HTTP. The difference is feature set vs price.

### HTTP API

The newer, simpler, cheaper option. AWS built it because most people were using REST APIs but only needed a fraction of the features.

- About **70% cheaper** than REST API
- Lower latency
- Native JWT authorizer support — works with Cognito, Auth0, any OIDC provider out of the box
- Built-in CORS support
- Supports Lambda and HTTP proxy integrations

**What it's missing:** no usage plans, no API keys, no request/response transformation, no response caching, no WAF integration at the API level, no per-route throttling.

### REST API

The original, more powerful option.

- Usage plans and API keys — throttle per customer, enforce quotas
- Request and response transformation — reshape the payload without touching the backend
- Response caching — cache backend responses for N seconds
- WAF integration
- Per-route throttling
- Stage variables — deploy the same API to dev/staging/prod with different config

More expensive and slightly more complex to configure.

### How to choose

| You need this | Pick this |
|---|---|
| Simple Lambda or HTTP backend with JWT auth | HTTP API |
| Per-customer throttling or API keys | REST API |
| Request/response transformation | REST API |
| Response caching | REST API |
| WAF integration | REST API |
| Lowest cost and latency | HTTP API |
| Bedrock agent streaming | REST API (longer timeouts + streaming support) |

A good rule of thumb: **start with HTTP API**. Switch to REST API only when you hit a specific feature wall. Most internal apps and straightforward backends never need REST API.

---

## Topic 2 — Throttling and Usage Plans

### Throttling

Throttling is rate limiting — controlling how many requests hit your backend per second. When a request is throttled, API Gateway returns **HTTP 429 Too Many Requests** immediately. Your Lambda never gets called.

Two settings control throttling:

| Setting | What it means |
|---|---|
| **Rate** | Steady-state maximum requests per second |
| **Burst** | Maximum spike above the rate limit |

AWS uses a **token bucket** algorithm. Tokens accumulate up to the burst limit when traffic is low. When traffic spikes, the bucket drains. Once empty, requests are throttled until it refills at the rate limit.

**Example:** rate is 100 req/s and burst is 500. A client can fire 500 requests instantly — draining the bucket. After that it's limited to 100/s while the bucket refills.

Throttling applies at multiple levels:

- **Account level** — default 10,000 req/s across all APIs in the region (soft limit, can be raised)
- **Stage level** — a ceiling for the whole API
- **Route level** — per-endpoint throttling (REST API only)

### Usage Plans (REST API only)

Usage plans let you control access per consumer. You create an API key, attach it to a usage plan, and that plan defines the throttle and quota for that key.

| Setting | What it controls |
|---|---|
| **Throttle** | Rate and burst limit for this specific key |
| **Quota** | Total requests allowed per day, week, or month |

When a quota is exhausted, API Gateway returns 429 — no Lambda invocation, no backend call.

**Example:** you're building a weather data API. Free tier gets 500 requests/day, pro tier gets 50,000/day. Each customer gets an API key assigned to the right plan. When a free tier customer exhausts their daily quota, they get a 429 — no backend code change needed, no logic in your Lambda.

---

## Topic 3 — Custom Domains

By default, API Gateway gives you a URL like:

```
https://a1b2c3d4e5.execute-api.us-west-2.amazonaws.com/prod/
```

A custom domain maps your own domain (`api.example.com`) to that URL. Under the hood, AWS creates a Regional endpoint in front of your API, and you point your DNS at it.

### Setup steps

1. Create a certificate in ACM — must be in the **same region** as your API for Regional endpoints
2. Create a Custom Domain Name in API Gateway and attach the cert
3. Create an **API mapping** — which API and stage lives at which path prefix
4. Add a **CNAME** in Route 53 pointing your domain to the API Gateway domain name AWS provides

### Path-based routing across multiple API

You can map multiple APIs to a single custom domain using path prefixes — no load balancer required.

| Path | API |
|---|---|
| `api.example.com/orders` | Orders API |
| `api.example.com/inventory` | Inventory API |
| `api.example.com/payments` | Payments API |

One domain, three separate APIs, each deployed and versioned independently. Clean for the client, no extra infrastructure needed.

---

## Topic 4 — mTLS (Mutual TLS)

### Normal TLS vs mTLS

**Normal TLS:** the client verifies the server's certificate. The server doesn't check the client. This is what every HTTPS connection does by default.

**mTLS:** both sides verify each other. The client presents a certificate. The server checks it against a truststore. If the client certificate isn't trusted, the connection is rejected — before any HTTP request is processed, before any Lambda is invoked.

### How it works in API Gateway

You upload a truststore — a `.pem` file containing the CA certificates you trust — to S3, then reference it in your custom domain config. API Gateway handles certificate verification at the TLS layer automatically. No Lambda code needed.

### When to use it

**Use mTLS for:**
- Machine-to-machine APIs where you control both sides
- Internal service-to-service calls between your own systems
- Partner integrations where you provision certificates to the partner

**Example:** a payment processor calls your webhook endpoint to notify you of successful charges. You issue them a client certificate from your own CA. Your API Gateway only accepts connections presenting that certificate. Even if someone discovers the endpoint URL, they can't call it without the cert.

**Do not use mTLS for:**
- Any API accessed from a browser or mobile app — browsers don't carry client certificates
- Public APIs with many consumers — managing and rotating certs at scale becomes a maintenance burden

For browser and mobile clients, use Cognito or JWT authorizers instead.

---

## Topic 5 — Cognito and JWT Authorizers

### The problem they solve

You need to know *who* is calling your API before your Lambda runs. Without an authorizer, any request with the right URL gets through. An authorizer checks the caller's identity and either lets the request proceed or returns a 401 — without your Lambda ever being invoked.

### JWT Authorizer (HTTP API)

The simplest option. You configure the issuer URL and the audience. When a request comes in, API Gateway:

1. Extracts the `Authorization: Bearer <token>` header
2. Fetches the public keys (JWKS) from the issuer
3. Validates the token signature
4. Checks expiry and audience
5. If valid — passes the request to your Lambda with the decoded claims in the event
6. If invalid — returns 401 immediately

You write zero validation code. API Gateway does it all. The issuer can be Cognito, Auth0, Okta, or any OIDC-compliant provider.

**Example:** a user logs into your app via Auth0 and gets a JWT. Every API call includes that token in the `Authorization` header. API Gateway validates it on every request. Your Lambda only ever sees authenticated traffic.

### Cognito Authorizer (REST API)

Same concept with tighter Cognito integration. You reference a Cognito User Pool directly rather than an OIDC endpoint. Cognito issues tokens after login; the client passes the ID token or access token on each request; API Gateway validates it against the User Pool automatically.

### Lambda Authorizer (both)

When JWT/Cognito isn't enough — for example, you need to check a database, validate a custom API key format, or implement logic that goes beyond standard token validation. You write a Lambda that receives the request (headers, path, method) and returns an IAM policy — `Allow` or `Deny`. If `Allow`, the request proceeds. If `Deny`, API Gateway returns 403.

**Example:** you're building a B2B SaaS API. Customers use custom API keys that you generate and store in DynamoDB — not JWTs. A Lambda authorizer receives the key, looks it up in DynamoDB, checks if it's active and not expired, and returns Allow or Deny.

**Important:** Lambda authorizers add latency because they invoke a Lambda on every request. Always enable the **authorizer cache** — responses are cached by token for a configurable TTL, so repeated calls with the same token don't re-invoke the authorizer Lambda.

---

## Topic 6 — Streaming Agent Responses

### Why this matters for Bedrock

When you invoke a Bedrock Agent, the response doesn't arrive all at once. The agent reasons, calls tools, observes results, reasons again — and the final answer streams token by token. If you wait for the full response before sending anything to the client, you might be waiting 20–30 seconds. That's a poor user experience.

You want to stream — send tokens to the client as they're generated, the same way ChatGPT shows text appearing word by word.

### Option A — Server-Sent Events (SSE)

One-way streaming from server to client over a regular HTTP connection. The client opens a connection and the server pushes data as it's ready. Natively supported by browsers with the `EventSource` API.

- Works over standard HTTP — no special protocol
- Simple to implement
- **One direction only** — server pushes to client, client cannot send mid-stream

### Option B — WebSockets

Full-duplex — both client and server can send messages at any time on the same persistent connection. API Gateway WebSocket APIs use a route-based model: you define routes (`$connect`, `$disconnect`, `$default`) and Lambda handlers for each.

- Necessary when the client needs to send messages mid-stream — interrupt a response, send a follow-up before the current one finishes
- More complex to set up — connection state needs to be stored in DynamoDB because each WebSocket message is a separate Lambda invocation
- Better for real-time bidirectional scenarios

### The 29-second timeout — the most important constraint

**API Gateway has a hard maximum integration timeout of 29 seconds.** Both REST API and HTTP API have this limit. It cannot be raised. If your backend hasn't sent a complete response within 29 seconds, API Gateway kills the connection and returns a **504 Gateway Timeout** to the client.

For a Bedrock Agent that calls multiple tools and generates a long response, 29 seconds is easy to exceed.

### How to work around the 29-second limit

| Approach | How it works | Best for |
|---|---|---|
| **WebSocket API** | No 29s timeout on WebSocket connections. Lambda pushes chunks back to the client using the `@connections` callback API as they arrive | Streaming Bedrock agents to a browser |
| **Async + polling** | Client gets a job ID immediately, polls a status endpoint until the response is ready | Simpler UX requirements, non-interactive use |
| **Direct SDK (backend only)** | Skip API Gateway for the streaming call, invoke Bedrock directly from your backend service | Server-to-server, no browser client involved |

### Which to pick

For **Bedrock agent streaming to a browser**: WebSocket API is the right answer. SSE through API Gateway hits the 29s limit. WebSocket connections bypass it, and the `@connections` model fits the streaming pattern cleanly.

For **simple non-streaming responses** under 29 seconds: HTTP API with a regular request/response — no streaming needed, keep it simple.

---

## Summary

| Topic | Key thing to remember |
|---|---|
| **HTTP API vs REST API** | HTTP API for most things — cheaper and simpler. REST API only when you need usage plans, caching, or transformation |
| **Throttling** | Rate = steady state req/s. Burst = spike headroom via token bucket. Returns 429 when exceeded |
| **Usage plans** | Per-customer throttle + quota. REST API only. Needed when different consumers need different limits |
| **Custom domains** | Map your domain to the API GW URL. One domain, multiple APIs via path prefix mapping |
| **mTLS** | Both sides verify certificates. For machine-to-machine only. Not for browser clients |
| **JWT authorizer** | API GW validates the token. Your Lambda gets the decoded claims. Zero auth code to write |
| **Lambda authorizer** | Custom auth logic in code. Always enable result caching to avoid per-request Lambda invocations |
| **SSE vs WebSockets** | SSE is simpler but hits the 29s timeout. WebSockets bypass it |
| **The 29s limit** | The single most important API GW constraint for async and streaming workloads |

---

*Day 8 — API Gateway notes. Part of the ongoing AWS + DevOps study series.*