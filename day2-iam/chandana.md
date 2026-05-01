# Day 2 — Chandana (AWS)

Notes, labs, and experiments for Day 2.

Picking up from the [Day 2 topics](./README.md) — focusing on the AWS-heavy ones:

- [ ] IAM core model — users, roles, policies, principals, request evaluation
- [ ] Writing least-privilege policies — Action / Resource / Condition keys
- [ ] Cross-account roles and STS AssumeRole

## Notes

_to be filled in as I go_

# AWS IAM 

# Topic 1 — IAM core model

---

## The five building blocks

| Concept | What it is | One-line rule |
|---|---|---|
| Principal | Who is making the request | User, role, AWS service, or federated identity |
| User | Long-lived identity with password / access keys | For humans — avoid in favor of roles |
| Role | Temporary identity anything can assume | For code, services, cross-account access |
| Policy | JSON document defining what is allowed or denied | Attached to a principal or a resource |
| Resource | The AWS thing being accessed, identified by ARN | `arn:aws:s3:::my-bucket`, `arn:aws:lambda:...:function:fn` |

---

## Policy anatomy

Every policy is a list of statements. Each statement has five fields:

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123:role/dev" },
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/uploads/*",
  "Condition": {
    "Bool": { "aws:SecureTransport": "true" }
  }
}
```

| Field | What it does | Key detail |
|---|---|---|
| `Effect` | Allow or Deny | Explicit Deny always wins over any Allow |
| `Principal` | Who the statement applies to | Only in resource-based policies — omit from identity-based |
| `Action` | Which API calls | `service:Operation`. Wildcards: `s3:Get*`, `s3:*`, `*` |
| `Resource` | Which specific AWS resources | By ARN. `*` = everything (dangerous for write/delete) |
| `Condition` | Extra constraints at request time | MFA, IP, region, tags, HTTPS — all optional but powerful |

**Two policy types:**
- Identity-based → attached to a user or role. No `Principal` field.
- Resource-based → attached to a resource (S3 bucket, Lambda, role). Has a `Principal` field.

---

## Request evaluation — how AWS decides Allow or Deny

Every API call goes through this in order. First match wins.

```
1. Explicit Deny anywhere?          → DENY immediately (no override possible)
2. SCP (org-level) allows?          → DENY if SCP doesn't permit it
3. Permissions boundary allows?     → DENY if boundary excludes it
4. Resource-based policy allows?    → ALLOW
5. Identity-based policy allows?    → ALLOW
6. Nothing said Allow?              → DENY (implicit default)
```

**Two rules that govern everything:**
1. AWS starts from deny. Silence is never permission.
2. An explicit Deny overrides every Allow — from any policy, at any level. The only way around a Deny is to remove it, or use `NotPrincipal` to carve out an exemption.

**Implicit vs explicit Deny:**
- Implicit = no policy said Allow → denied by default → can be overridden by an Allow
- Explicit = someone wrote `"Effect": "Deny"` → cannot be overridden by any Allow

---

## Users vs roles — the key difference

| | IAM user | IAM role |
|---|---|---|
| Credentials | Long-lived (access key + secret) | Short-lived (STS temp creds, expire in 1–12h) |
| Who uses it | A person, long-term | Code, services, other accounts, federated users |
| Risk | High — leaked keys work forever | Low — credentials expire automatically |
| Correct use | Almost never in production | Everything: Lambda, EC2, CI/CD, cross-account |

> Rule: if code needs AWS access, it gets a role. Users are for humans signing into the console, and even then SSO → role is better.

---

# Topic 2 — Writing least-privilege policies

---

## Action — control exactly which API calls are allowed

Format: `service:OperationName`. Every SDK call maps to exactly one action.

```json
"Action": "s3:GetObject"                     // single action
"Action": ["s3:GetObject", "s3:PutObject"]   // list
"Action": "s3:Get*"                           // all S3 read actions
"Action": "s3:*"                              // all S3 actions (usually too broad)
"Action": "*"                                 // everything — never in production
```

**Least-privilege approach:** look at your code, find every SDK call, list only those actions.

```json
// Lambda reading from S3 — correct
"Action": ["s3:GetObject", "s3:ListBucket", "logs:CreateLogGroup",
           "logs:CreateLogStream", "logs:PutLogEvents"]

// Do NOT use
"Action": "s3:*"   // grants Delete, ACL changes, bucket deletion too
```

**`NotAction`** — inverse. Apply statement to all actions *except* listed. Use with `Effect: Deny` to enforce hard limits.

---

## Resource — scope to specific ARNs

ARN format: `arn:aws:service:region:account-id:resource`

```bash
# S3 — no region or account (globally unique names)
arn:aws:s3:::my-bucket          # the bucket itself
arn:aws:s3:::my-bucket/*        # all objects in the bucket
arn:aws:s3:::my-bucket/uploads/*  # only the uploads/ prefix

# Lambda
arn:aws:lambda:us-west-2:123456789012:function:my-fn

# IAM role (global — no region)
arn:aws:iam::123456789012:role/my-role
```

**S3 gotcha — ListBucket vs GetObject need different scopes:**
```json
[
  {
    "Effect": "Allow",
    "Action": "s3:ListBucket",
    "Resource": "arn:aws:s3:::my-bucket"       // bucket ARN (no /*)
  },
  {
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"      // objects ARN (with /*)
  }
]
```
Swapping these scopes causes silent AccessDenied. This is the most common S3 IAM mistake.

**Policy variables** — dynamic scoping without hardcoding:
```json
"Resource": "arn:aws:s3:::home-bucket/${aws:username}/*"
// User "chandana" → home-bucket/chandana/* only
// User "alice" → home-bucket/alice/* only
// One policy, per-user scope
```

---

## Condition — extra constraints at request time

Structure: `{ "Operator": { "ConditionKey": "value" } }`. All conditions in a block must be true (AND logic).

**The most useful conditions:**

```json
// Require HTTPS — deny plain HTTP
"Condition": { "Bool": { "aws:SecureTransport": "false" } }  // on a Deny statement

// Require MFA — only allow destructive actions with MFA
"Condition": { "Bool": { "aws:MultiFactorAuthPresent": "true" } }  // on an Allow statement

// Lock to specific region — data residency
"Condition": { "StringEquals": { "aws:RequestedRegion": "ap-south-1" } }

// Restrict to IP range — office access only
"Condition": { "IpAddress": { "aws:SourceIp": "203.0.113.0/24" } }

// ABAC — tag-based access control
"Condition": {
  "StringEquals": {
    "aws:ResourceTag/team": "${aws:PrincipalTag/team}"
  }
}
// Principal tagged team=backend → can only touch resources tagged team=backend
// One policy handles thousands of resources without naming any ARN
```

**Condition operators quick reference:**

| Operator | Use for |
|---|---|
| `StringEquals` / `StringNotEquals` | Exact string match |
| `StringLike` | Wildcard match (`*` and `?`) |
| `Bool` | true/false values |
| `IpAddress` / `NotIpAddress` | CIDR ranges |
| `ArnLike` / `ArnEquals` | ARN matching |
| Add `IfExists` suffix | Make condition pass when key is absent |

---

## Complete least-privilege policy example

Lambda function that reads from S3 and writes CloudWatch logs:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::my-app-bucket"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-app-bucket/data/*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": ["arn:aws:s3:::my-app-bucket", "arn:aws:s3:::my-app-bucket/*"],
      "Condition": { "Bool": { "aws:SecureTransport": "false" } }
    }
  ]
}
```

---

# Topic 3 — Cross-account roles and STS AssumeRole

---

## The problem it solves

Account A (dev/tooling) needs to access resources in Account B (prod). You never copy credentials between accounts. Instead, Account A's code temporarily becomes a role in Account B using STS.

---

## The AssumeRole flow

```
Account A                    AWS STS                  Account B
─────────                    ───────                  ─────────
Lambda/EC2
(has caller role)
    │
    │── 1. sts:AssumeRole ──────────────────────────────────────▶
    │       + ExternalId                              (validates trust policy)
    │
    │◀─ 2. Returns temp creds ────────────────────────────────────
    │       AccessKeyId
    │       SecretAccessKey
    │       SessionToken (expires 15min–12h)
    │
    │── 3. Use temp creds to call Account B APIs directly ───────▶
                                                      Account B
                                                      target role
                                                      (has permissions)
```

**Two policies must both exist and agree:**
1. Caller (Account A) identity policy: allows `sts:AssumeRole` on the target role ARN
2. Target role (Account B) trust policy: trusts the caller role ARN

---

## The two required policies

**Account B — trust policy on the target role:**
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::111111111111:role/caller-role"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "my-shared-secret-42"
      }
    }
  }]
}
```

**Account A — identity policy on the caller role:**
```json
{
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::222222222222:role/target-role"
  }]
}
```

---

## The code (Python / boto3)

```python
import boto3

sts = boto3.client("sts")

# Step 1 — assume the role, get temp credentials
resp = sts.assume_role(
    RoleArn="arn:aws:iam::222222222222:role/target-role",
    RoleSessionName="chandana-job-20240115",   # shows in CloudTrail — name it meaningfully
    ExternalId="my-shared-secret-42",           # must match Account B's trust policy
    DurationSeconds=3600                         # 1 hour, max 12h
)

creds = resp["Credentials"]

# Step 2 — use temp credentials to create a new client in Account B
s3 = boto3.client("s3",
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"]      # required for temp creds
)

# Now s3 calls operate as the target role in Account B
s3.get_object(Bucket="prod-bucket", Key="data/report.csv")
```

---

## ExternalId — the confused deputy protection

Without `ExternalId`, any role in Account A can trick Account B by knowing the target role ARN. `ExternalId` is a shared secret that Account B requires in the assume request. Even if an attacker knows the ARN, they don't know the ExternalId.

Always use `ExternalId` for third-party or multi-tenant cross-account access.

---

## RoleSessionName — audit trail

Every action taken with assumed-role credentials appears in CloudTrail as:
```
arn:aws:sts::222222222222:assumed-role/target-role/chandana-job-20240115
```

Name the session after the job, user, or service — not a random string. This is how you debug "who deleted that file in prod?" six months later.

---

## Common patterns

| Pattern | How |
|---|---|
| Lambda in Account A reading prod S3 in Account B | Lambda role → AssumeRole → Account B S3 reader role |
| CI/CD pipeline deploying to prod | GitHub Actions OIDC → AssumeRole → prod deploy role |
| Centralized logging account | All accounts assume a role in the logging account to ship logs |
| Multi-tenant SaaS | One role per customer account, ExternalId = customer ID |

---

# Quick reference

```
IAM CORE RULES
  Default = deny. Silence is never permission.
  Explicit Deny wins over every Allow, from any policy.
  Implicit deny (nothing said Allow) CAN be overridden by an Allow.
  Explicit deny (someone wrote "Effect": "Deny") CANNOT be overridden.

PRINCIPAL TYPES
  User         → long-lived creds, for humans, avoid in prod
  Role         → temp creds, for everything else
  AWS service  → Lambda, EC2 etc. — always use a role
  Federated    → SAML/OIDC login mapped to a role

EVALUATION ORDER
  Explicit Deny → SCP → Permissions boundary → Resource policy → Identity policy → Implicit deny

LEAST-PRIVILEGE CHECKLIST
  Action   → list specific operations, never use * in production
  Resource → specific ARNs, never * for write/delete
  Condition → add aws:SecureTransport, MFA, region lock as needed
  S3 rule  → ListBucket on bucket ARN, GetObject/PutObject on bucket/* ARN

STS ASSUMEROLE
  Caller needs sts:AssumeRole on target role ARN (identity policy)
  Target role needs trust policy naming the caller ARN
  Returns: AccessKeyId + SecretAccessKey + SessionToken (expires)
  Always use ExternalId for cross-account third-party access
  Name the RoleSessionName for CloudTrail auditability
```
