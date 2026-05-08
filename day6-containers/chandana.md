# Day 6 — Chandana (AWS)

Notes, labs, and experiments for Day 6.

## Topics
- [x] ECS vs EKS vs Fargate — when to pick which, and what each one really costs to operate
- [x] Task definitions, services, and ALB integration on ECS
- [x] Autoscaling — target tracking vs step scaling, scaling on custom metrics

---

## The big picture — how it all connects

```
Your container image (ECR)
  ↓
Task Definition         ← blueprint: CPU, memory, ports, secrets, IAM
  ↓
Service                 ← keeps N tasks running, handles deploys
  ↓
Cluster (ECS or EKS)    ← orchestrator: decides where tasks run
  ↓
Compute (EC2 or Fargate) ← actual machines that run your containers
  ↓
ALB                     ← routes internet traffic to healthy tasks
  ↓
Autoscaling             ← adjusts task count based on load
```

---

## 1. ECS vs EKS vs Fargate

### Understand the relationship first

These are not three alternatives — they are two separate decisions.

| Decision | Options |
|---|---|
| **Orchestrator** (who manages containers) | ECS (AWS-native) or EKS (Kubernetes) |
| **Compute** (where containers actually run) | EC2 (you manage) or Fargate (AWS manages) |

Most teams use **ECS + Fargate** — AWS orchestration with serverless compute.

---

### ECS — AWS-native orchestrator

ECS is AWS's own container orchestration system. Simpler than Kubernetes, tighter AWS integration, no extra control plane cost.

Four building blocks — each builds on the previous:

| Concept | What it is |
|---|---|
| **Cluster** | Logical boundary — all tasks and services live inside one cluster |
| **Task definition** | Blueprint — describes what to run, how much CPU/memory, which ports, which IAM role |
| **Task** | A running instance of a task definition — ephemeral, starts and stops |
| **Service** | Keeps N tasks running at all times — replaces crashed tasks, handles rolling deploys |

**Use ECS when:**
- You are on AWS and don't have Kubernetes expertise
- You want simplicity and fast setup
- You want native AWS integration without extra tooling

---

### EKS — Kubernetes on AWS

EKS gives you a fully managed Kubernetes control plane. The API is standard Kubernetes — everything in the Kubernetes ecosystem works as-is.

ECS → Kubernetes mapping:

| ECS | EKS (Kubernetes) |
|---|---|
| Task definition | Pod spec |
| Task | Pod |
| Service | Deployment + Service |
| Cluster | Cluster + Namespace |

**Use EKS when:**
- Your team already knows Kubernetes
- You run on multiple clouds and need portability
- You need Kubernetes-specific tools (Istio, Argo CD, Keda)
- You have hundreds of services needing advanced scheduling

**EKS costs $0.10/hr = ~$73/month just for the control plane** — before any compute.

---

### Fargate — serverless compute layer

Fargate removes EC2 management entirely. You say "run this task with 0.5 vCPU and 1GB memory" — AWS finds the compute, runs it, you pay only while the task is running.

**What you gain:** no instance provisioning, no OS patching, no cluster capacity management

**What you give up:**

| Limitation | Detail |
|---|---|
| No SSH to host | Use ECS Exec to shell into the container instead |
| Higher per-unit cost | You pay a premium for the managed experience |
| No GPU support | GPU workloads must use EC2 |
| Container coldstart | ~10–30s for a new task to start (EC2 instances are already warm) |

---

### Cost comparison — 10 always-on containers (1 vCPU / 2GB each)

| Setup | Monthly cost | Operational overhead |
|---|---|---|
| ECS + Fargate | ~$280 | Minimal |
| ECS + EC2 On-Demand | ~$150 | Medium — patch, size, manage instances |
| ECS + EC2 Spot | ~$50 | High — handle interruptions |
| EKS + Fargate | ~$353 | Medium — $73 control plane + compute |
| EKS + EC2 | ~$223 | High |

> For most teams at early/mid scale — **ECS + Fargate** wins on total cost including engineering time.

---

### Decision guide

```
No Kubernetes expertise, on AWS?
  └── ECS

Need Kubernetes ecosystem / multi-cloud / already run K8s?
  └── EKS

For either — what compute?
  ├── Simple workloads, want zero instance management  → Fargate
  ├── Need GPU (ML inference)                         → EC2
  ├── High scale, cost-sensitive, have ops capacity   → EC2 Spot
  └── Optimise base cost, burst on demand             → EC2 + Fargate mixed
```

---

## 2. Task Definitions, Services, and ALB Integration

### Task definitions — the blueprint

A task definition is a versioned JSON document. Every change creates a new revision (`my-app:1`, `my-app:2`). Services pin to a revision or always use latest.

**Key things you configure:**

**CPU and memory** — set at task level (total budget). On Fargate, not all combinations are valid:

| CPU | Valid memory range |
|---|---|
| 0.25 vCPU | 0.5GB – 2GB |
| 0.5 vCPU | 1GB – 4GB |
| 1 vCPU | 2GB – 8GB |
| 2 vCPU | 4GB – 16GB |
| 4 vCPU | 8GB – 30GB |

**Two IAM roles — always configure both:**

| Role | Purpose |
|---|---|
| **Task execution role** | What ECS needs to *start* the task — pull image from ECR, write logs, fetch secrets |
| **Task role** | What your *application code* can do at runtime — call S3, invoke Bedrock, write to DynamoDB |

**Secrets** — never put secrets in plain environment variables. Reference them from Secrets Manager in the task definition — ECS fetches and injects at container start.

**Health check** — a command ECS runs inside the container to decide if it is healthy. Failed health check → ECS replaces the task. Set a `startPeriod` (e.g. 60s) so slow-starting apps aren't killed before they are ready.

**Network mode** — always `awsvpc` on Fargate. Each task gets its own private IP. Tasks talk to each other via private IP or service discovery, not via localhost.

---

### Services — keeping tasks running

A service wraps a task definition and does four things:

**1. Maintains desired count** — you say `desiredCount: 3`, ECS always keeps 3 tasks running. Task crashes → ECS launches a replacement immediately.

**2. Rolling deployments** — when you push a new image, the service replaces old tasks with new ones gradually. Two parameters control this:

| Parameter | Meaning |
|---|---|
| `minimumHealthyPercent` | Floor of running tasks during deploy. 100% = no capacity reduction |
| `maximumPercent` | Ceiling during deploy. 200% = run double tasks temporarily (half old, half new) |

Production default: `minimumHealthyPercent: 100`, `maximumPercent: 200` → zero-downtime rolling deploy.

**3. Load balancer integration** — registers new tasks with the ALB target group when they start, deregisters them when they stop. Existing connections are allowed to finish (connection draining, default 300s).

**4. Service discovery** — optionally registers tasks in AWS Cloud Map so other services find them by DNS name rather than hardcoded IPs.

---

### ALB integration — how traffic reaches containers

```
Internet
  ↓
Route 53 → ALB DNS name
  ↓
ALB Listener (port 443, TLS terminated here)
  ↓ listener rules: route by host or path
Target Group  ← ECS registers/deregisters tasks here automatically
  ↓
Container (plain HTTP inside your VPC)
```

**ALB components:**

| Component | What it does |
|---|---|
| **Listener** | ALB listens on port 80 or 443. Each listener has routing rules |
| **Listener rules** | Route by host header, path, method, or query string |
| **Target group** | Pool of tasks receiving traffic. Runs health checks. Unhealthy tasks removed from rotation |

**Two health checks — set both:**

| Health check | Who runs it | Failure consequence |
|---|---|---|
| ECS container health check | ECS | ECS replaces the task |
| ALB target group health check | ALB | ALB stops sending requests to that task |

**HTTPS termination** — ALB handles TLS. Your containers receive plain HTTP. Certificate lives in ACM (AWS Certificate Manager), attached to the ALB listener. No cert management inside containers.

**Path-based routing — one ALB, multiple services:**

```
ALB (one public IP, one certificate, ~$18/month)
  ├── /api/*     → Target Group: api-service
  ├── /admin/*   → Target Group: admin-service
  └── /*         → Target Group: frontend-service
```

One ALB shared across services saves ~$30–45/month vs running separate ALBs per service.

---

## 3. Autoscaling

### Three layers — understand which one you are configuring

| Layer | What scales | Applies to |
|---|---|---|
| **Task autoscaling** | Number of tasks in a service | ECS + Fargate and ECS + EC2 |
| **Capacity provider** | Number of EC2 instances in the cluster | ECS + EC2 only |
| **Application level** | Connection pools, queue workers inside your app | Always |

On Fargate you only deal with **task autoscaling** — Fargate handles compute capacity automatically.

---

### Target tracking — start here, use this 90% of the time

You pick a metric and a target value. AWS automatically adds or removes tasks to keep the metric at that target. Like a thermostat — set the target temperature, the system handles heating and cooling.

**Common target metrics:**

| Metric | Target | Use when |
|---|---|---|
| CPU utilization | 60–70% | General purpose — most services |
| Memory utilization | 60–70% | Memory-bound workloads (JVM, ML) |
| ALB requests per target | depends on app | HTTP services where CPU doesn't reflect load |

**Why 60–70% and not 90%?** — New tasks take ~60 seconds to start. If you wait until 90% CPU to scale, you are already saturated before new capacity arrives. Target 60–70% to stay ahead of the spike.

**Scale-out is aggressive, scale-in is conservative** — by design. AWS adds tasks quickly when the metric rises, but waits (default 300s cooldown) before removing tasks when it drops. This prevents oscillation.

---

### Step scaling — when you need more control

Step scaling lets you define exactly what happens at different threshold levels. You define bands and actions:

```
CPU < 40%   → remove 2 tasks   (aggressively scale in when idle)
CPU 40–60%  → do nothing       (steady state)
CPU 60–75%  → add 1 task
CPU 75–85%  → add 2 tasks
CPU > 85%   → add 4 tasks      (scale hard when near saturation)
```

Step scaling requires a CloudWatch alarm to trigger it.

**Use step scaling when:**
- You want asymmetric scaling (scale in slowly, scale out fast)
- You need to add a large burst of capacity at a specific threshold
- Target tracking alone is too slow for sudden spikes

In practice — start with target tracking. Add a step scaling policy as a safety net for sudden spikes that target tracking is too slow to handle.

---

### Scaling on custom metrics

Both policies can use **any CloudWatch metric** — including metrics your application publishes.

| Custom metric | Scaling target | Pattern |
|---|---|---|
| SQS queue depth | 10 messages per task | Job worker — scale with backlog size |
| p99 request latency | your SLA threshold | Scale before users feel it |
| Active user sessions | sessions per task | Scale with actual user load |
| Order rate | orders per task | Scale with business volume |

This is more accurate than CPU for many workloads — CPU doesn't always correlate with what's actually stressing your service.

---

### Scaling limits — always set these

| Setting | What it does | Recommendation |
|---|---|---|
| **Minimum tasks** | Floor — never scale below this | At least 2 in production (single task = single point of failure) |
| **Maximum tasks** | Ceiling — never scale above this | Set to what you are comfortable paying at peak |
| **Scale-in protection** | Individual task marked safe from removal | Use for tasks mid-way through a long job |

---

### Cooldowns — prevent oscillation

| Cooldown | Default | Recommendation |
|---|---|---|
| Scale-out cooldown | 300s | Shorten to 60–120s — scale out fast |
| Scale-in cooldown | 300s | Keep at 300–600s — scale in conservatively |

Without cooldowns: metric spikes → add task → metric drops → remove task → metric spikes → repeat forever.

---

### A production scaling setup

```
Service: api-production
  Desired: 2 tasks
  Minimum: 2          ← never below 2 (high availability)
  Maximum: 20         ← cost ceiling

  Policy 1 — Target tracking (day-to-day scaling)
    Metric: CPU utilization
    Target: 65%
    Scale-out cooldown: 120s
    Scale-in cooldown: 300s

  Policy 2 — Step scaling (safety net for sudden spikes)
    Alarm: CPU > 85% for 1 minute
    Action: add 4 tasks immediately

  Two policies on one service — AWS always takes the most aggressive action.
```