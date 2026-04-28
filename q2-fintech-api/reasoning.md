# Q2 — How I Thought Through This (Step-by-Step)

This is the working-out for Q2 (Fintech API). The polished answer is in `essay.md`; here I capture the thinking, including bits I'm unsure about.

---

## Step 1 — Split the question into three sub-problems

Q2 felt huge at first. I split it into three sub-problems that match the three Tasks:

1. **Architecture** (Task 1) — what AWS services run what, where customer data lives, how it scales for payday.
2. **SSO for mobile** (Task 2) — one login, all endpoints.
3. **Geo-fraud detection** (Task 3) — block users by location using Google Maps history.

These three are mostly independent. I solved them in this order because each builds context for the next.

---

## Step 2 — Identify the hard constraints

Same approach as Q1: list what I cannot trade away.

| Hard constraint | What it forces |
|---|---|
| Customer data must stay in Thailand (PDPA) | Customer DB lives **on-prem**, not in the cloud |
| API must always be available (req 2.1) | Multi-AZ at minimum |
| API must handle ultra-high payday load (req 2.2) | Auto-scaling required |
| Detect Eavesdropping, SQLi, DDoS (req 6.1) | WAF + Shield + TLS — these are not optional |
| One backup must be super-secure (req 5) | Same 3-2-1-1-0 pattern as Q1 |

Once I had this, the rest of the design fell out naturally.

---

## Step 3 — The compute choice: EC2 Auto Scaling Group

This is the most important architectural decision. I weighed three options:

| Option | Pros | Cons |
|---|---|---|
| **EC2 Auto Scaling group + ALB** | Same pattern as Q1 — easy to compare; full control of the runtime; fits the course's classical web-app model | Slower to scale (minutes, not seconds); pay for instances even when idle |
| AWS Lambda | Auto-scales instantly. Pay per request. No servers to patch. | Cold starts; 15-min execution limit; different mental model from Q1 |
| ECS Fargate (containers) | Predictable performance, no cold starts | More setup than EC2 ASG; not in the course's main material |

**My choice: EC2 Auto Scaling group + ALB**, for two reasons:

1. **Consistency with Q1.** The course teaches this pattern as the standard cloud architecture, and using it across the questions makes the videos easier to record and the marker easier to read.
2. **Predictable scaling for predictable spikes.** Payday is a *known* event. With **scheduled auto-scaling**, I scale up *before* the spike — there's no need for the instant scaling Lambda would offer.

**Trade-off accepted**: EC2 ASG takes 2–3 minutes to scale up unpredictably. Lambda would scale instantly. For payday traffic this doesn't matter (we know when payday is). For an unexpected spike, the higher minimum capacity buys time while the ASG adds instances.

### Where I still use Lambda

There are two spots where running EC2 doesn't make sense and a small Lambda is the right tool:

1. **Nightly cron** (req 4) — runs once a day for ~1 minute. EC2 would idle for the other 23 hours and 59 minutes.
2. **SOAR playbooks** (req 6.2) — short scripts triggered by EventBridge when a security alert fires. EC2 doesn't fit "run 30 lines of Python when an alarm fires".

So the rule is: **EC2 ASG for the API server; Lambda only for tiny event-triggered scripts.**

---

## Step 4 — Build the edge layer

The exam asks for protection against **Eavesdropping, SQL injection, DDoS**. These map onto AWS services directly:

- **Eavesdropping** → TLS 1.3 everywhere → CloudFront does this.
- **SQL injection** → AWS WAF managed rules + parameterised queries in EC2 code.
- **DDoS** → AWS Shield Advanced.

So the edge layer practically writes itself: **Route 53 → Shield → CloudFront → WAF → ALB → EC2**. Each layer has one job.

**Note on integration**: Route 53 and Shield are both *auto-integrated* with CloudFront:
- Route 53 automatically routes to CloudFront distributions — enabling CloudFront means Route 53 works with it.
- Shield is enabled *on* CloudFront distributions — it automatically protects all CloudFront endpoints without separate configuration. This is why we show them as separate boxes (to make each requirement visible), but in practice they're integrated services.

---

## Step 5 — Identity (Task 2)

This is a question I'm comfortable with as a software engineer. The standard pattern:

- **OAuth 2.0** for authentication.
- **JWT** for the bearer token.
- **The application** validates the JWT on every request.

The AWS service for the IDP is **Amazon Cognito**.

How does this give "one login, all endpoints"?
- The user logs in once → Cognito hands back a JWT.
- Every endpoint runs the same JWT-validation middleware → they all accept the same JWT.
- That's literally SSO across endpoints.

**Difference from a Lambda-based architecture**: in a Lambda design, API Gateway has a built-in Cognito authorizer that validates the JWT before invoking Lambda. In our EC2 design, the JWT validation runs as middleware **inside** the EC2 app code (a standard library does the signature check against Cognito's JWKS keys). Same security; just lives in a different place.

---

## Step 6 — Geo-fraud detection (Task 3)

The question: deny access if the user is not in Thailand or visited specific countries in the last 3 months, **using their Google Maps location history**.

I broke this down into:

1. How do we *get* the user's location history?
2. *When* do we check it?
3. What rules do we apply?
4. What happens when the data isn't available?

### Getting the data

Google Maps location history is per-user data owned by the user. We need their consent. Standard pattern: **OAuth**.

- The mobile app shows a Google sign-in screen with a scope request.
- On consent, Google returns an OAuth refresh token.
- We store it encrypted in **AWS Secrets Manager**, keyed by the user's Cognito ID.

### When do we check?

| Option | Pros | Cons |
|---|---|---|
| Once at login | Few Google calls | Doesn't catch users who travel mid-session |
| Every API call | Strongest enforcement | Many Google calls → rate limits + latency |
| **Cached, with TTL** | Balance | Adds caching logic |

I'm going with **cached** — store the latest decision in DynamoDB with a 30-minute TTL.

### The rules

Four rules: must be in Thailand, no visit to blocklisted countries in last 3 months, physically possible travel only, fallback to IP+GPS if Google data isn't available.

### Where does the fraud-check fit?

Since we're on EC2, the fraud-check is **middleware** in the application code. Order:
1. JWT validation middleware (proves *who*).
2. Fraud-check middleware (proves *where*).
3. Both pass → the route handler runs.

In a Lambda design this would be a separate Lambda authorizer in API Gateway. Same logic, different container.

**What I'm unsure about**:
- "Last 3" in the question — I assumed "last 3 months". Could be days. The video should call out the assumption.
- Google's exact API for Timeline data has changed over time. I'd verify against current Google docs.

---

## Step 7 — Nightly summary push (req 4)

Easy to miss this small piece. The on-prem accounting system needs nightly transaction summaries.

The pattern:
- **EventBridge Scheduler** at 23:00 daily → triggers the small `nightly-summary` Lambda.
- Lambda aggregates transactions into a summary file.
- **AWS DataSync** pushes the file over the VPN.

This is one of two places where Lambda is a better fit than EC2: a 1-minute job that runs once a day. Provisioning an EC2 instance for that would waste 99% of the day.

---

## Step 8 — Backups and IR (reqs 5 + 6.2)

Same pattern as Q1, lifted directly:

- 3-2-1-1-0 rule.
- AWS Backup → S3 with Object Lock.
- Storage Gateway → physical tape.
- IR framework: NIST SP 800-61.
- Detection: GuardDuty + Security Hub (GuardDuty auto-ingests CloudTrail, VPC Flow Logs, and Route 53 DNS logs — no manual wiring needed).
- Automated response: EventBridge → small Lambda playbooks (the second place where small Lambdas are right).

I'm reusing the same pattern deliberately — these are general-purpose, not bespoke per question.

---

## Step 9 — Why DynamoDB for the cloud data

I chose DynamoDB for three types of data in the cloud layer:

1. **Rate-limit counters**: Every API call needs to check "have they hit their limit?" — sub-millisecond latency required. DynamoDB does this natively. (Redis would be faster, but DynamoDB is already in the stack and sufficient.)

2. **Fraud-check cache**: Task 3 caches the geo-fraud decision for 30 minutes to avoid calling Google API every request. DynamoDB's built-in TTL feature auto-expires items — no cleanup code needed. Key-value access pattern is a perfect fit.

3. **Audit log**: Payday means millions of writes. DynamoDB on-demand scaling handles spike traffic without pre-provisioning. Each API request logs one record — write-heavy, time-series pattern suits DynamoDB.

**Bottom line**: DynamoDB handles the high-throughput, short-lived data. Aurora handles relational queries. The two together = optimal.

---

## Step 10 — Things I'd dig deeper on

Honest list:

1. **EC2 ASG scale-up timing** — for a real fintech, I'd model the exact payday traffic curve and tune the scheduled scale-up window so we never serve degraded.
2. **Cognito quotas** — for a payments app at scale, Cognito has soft quotas; would need an enterprise tier.
3. **Google Maps API reliability** — what happens when it's down? My fallback is reasonable but not load-tested.
4. **Multi-Region failover** — Route 53 failover works, but the DB story across regions needs care which I hand-waved.
5. **Storing third-party OAuth tokens** — there's compliance work here (consent records, deletion on user request).

---

## How this maps to what the marker is looking for

- **Mapping table** — every requirement → a named component.
- **Per-component justification** — why each piece is there.
- **Trade-offs explicit** — EC2 ASG vs Lambda, VPN vs Direct Connect, Cognito vs Auth0.
- **Task 2 has a flow diagram** — makes the OAuth answer concrete.
- **Task 3 has a rules table + failure modes** — answers the question's prompt directly.
- **Diagram** — single picture the marker can scan.

The compute choice (EC2 ASG instead of Lambda) is a deliberate consistency decision across questions, not a technical preference. The video should call this out: *"I'm using EC2 ASG here for consistency with Q1 — Lambda would also work; I picked EC2 because predictable payday spikes can be handled by scheduled scaling."*
