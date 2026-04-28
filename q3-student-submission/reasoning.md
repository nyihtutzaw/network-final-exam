# Q3 — How I Thought Through This (Step-by-Step)

The polished answer is in `essay.md`. This file captures the reasoning, including assumptions and uncertainty.

---

## Step 1 — Spot the design driver

The single most important phrase in the question is:

> ...get this system up and running quickly with **minimal server hardware upfront purchase and maintenance**.

That changes the whole shape of the answer. In Q1 (hospital) and Q2 (fintech) the hard constraints were *security* and *availability*. In Q3 the hard constraint is **speed of delivery**. Every architectural choice should be measured by "does this let the team ship faster?"

That immediately ruled out an EC2-based architecture for me. EC2 means VMs to patch, scale, and monitor — exactly the work the team wants to avoid.

The answer is **serverless-first**: lean on managed services (SaaS + PaaS) and pay AWS to run the boring parts.

---

## Step 2 — Map the cloud-service categories before naming services

Task 1 explicitly asks me to identify which **PaaS / SaaS / FaaS** is used for which purpose. So I worked it the other way around: I categorised each requirement *before* picking concrete services.

| Requirement | What kind of work is this? | Best category |
|---|---|---|
| Authentication, MFA | Solved problem, off-the-shelf | **SaaS** |
| Email | Solved problem | **SaaS** |
| Virus scanning, plagiarism AI | Specialist services | **SaaS** |
| Frontend hosting | Static files | **PaaS** |
| API platform | Routing + auth + throttling | **PaaS** |
| Real-time chat | WebSocket fan-out | **PaaS** |
| Database | Managed storage | **PaaS** |
| Business logic | The code we have to write | **FaaS** (so we don't manage runtimes) |

Then I picked the AWS service for each. For a senior SWE, this part is mechanical.

---

## Step 3 — Picking AWS Lambda for everything we write

Lambda is the single biggest decision. It's tempting to write "Lambda for everything", but I want to be honest about when I'd reach for something else:

- **Lambda fits when**: short request/response, event-driven work, glue code, scheduled jobs.
- **Lambda is awkward for**: anything > 15 minutes, very heavy CPU/RAM workloads, code that needs persistent connections to many clients.

For this platform, almost everything is Lambda-friendly. The two grey areas:

1. **Real-time chat.** A Lambda runs for a single request, but a chat connection needs to stay open. So I use **AWS AppSync**, which holds the live connection; Lambda only handles each message.
2. **Long-running plagiarism checks.** Most plagiarism APIs return in minutes, so Lambda is fine. If they took longer than 15 minutes I'd switch to AWS Step Functions.

---

## Step 4 — The file-upload flow (this took the most thought)

Requirement 3.2 says "large temporary storage" with "late submissions allowed for one month". I had to figure out:

- **Where do the bytes go?** Lambda has a 6 MB request limit and isn't a good place to receive large files. So I use **pre-signed S3 upload URLs**: the Lambda generates a temporary upload URL; the browser uploads to S3 directly.
- **How do we keep them for a month?** S3 lifecycle rules: keep in normal storage for 30 days (fast access for late submissions), then move to cheaper archive storage, delete after 60. Configured once, runs forever.
- **How do we trigger virus scan?** Every S3 upload triggers a Lambda automatically. AWS does this natively — one config line.

This is a great example of why serverless feels productive: the file flow has zero server code beyond the URL-generation Lambda. S3 + lifecycle + event triggers give me the rest.

**Trade-off accepted**: pre-signed URLs leak briefly through CloudFront/browser/network logs. For student projects (not bank statements) this is fine.

---

## Step 5 — Roles (Task 2)

Task 2 asks for the login + role-taking method. I considered three approaches:

| Approach | Pros | Cons |
|---|---|---|
| Build login + roles ourselves | Total control | Slow to build, easy to get wrong |
| **Cognito User Pool + Cognito Groups** | Managed login, MFA built in, JWT carries the role | Login UI is limited |
| Use the university's existing single-sign-on system | Students reuse their existing login | More setup, depends on IT to integrate |

I went with **Cognito + Groups**. It's a clean master's-level answer — standard login + role-based access. If the university wants single sign-on with their existing campus login, Cognito can federate with it later.

The role goes into the JWT in the `cognito:groups` claim. After that:
- API Gateway routes are scoped by group (students can't hit `/teacher/*`).
- Each Lambda checks the role and the record ownership.

This is the simplest model that satisfies "different roles and data permissions".

**What I'm less sure about**:
- How fine-grained should permissions be? The question says "different roles and data access permission" — that's vague. I went with two layers (route + record). For master's-level, that's enough.
- How are roles assigned at signup? I picked "by email domain" because it's pragmatic for a university. If their domains aren't clean, the team would need a manual admin tool.

---

## Step 6 — The "rapid development" theme

In the essay I made the cloud-service categories table explicit because that's literally what Task 1 asks for. But the deeper point is: **the team's job becomes glue code + UI**, because every undifferentiated piece is bought as a service.

When I write this in the video, the punch line is:

> If you tried to build this on bare EC2, you'd spend the first 6 weeks on auth, email, virus scan, and load balancing — none of which is the actual product. With this architecture, week 1 is auth and S3 upload; week 2 is a working MVP.

---

## Step 7 — Things I'd dig deeper on

Honest list:

1. **Cognito vs federated IDP** — for a real university the right answer is probably federation, not native Cognito. I went with native Cognito because it's simpler to explain and demo.
2. **ClamAV in Lambda** — works, but the signature database needs daily updates and the Lambda has cold-start cost. VirusTotal is simpler but paid.
3. **Plagiarism API selection** — Turnitin is the academic standard, Copyleaks is a popular cheaper alternative. The actual choice is procurement.
4. **DynamoDB schema design** — I hand-waved this. For real, I'd carefully model access patterns to avoid hot partitions.
5. **AppSync vs API Gateway WebSocket** — both can do real-time chat. AppSync is GraphQL-flavoured (easier subscriptions), API Gateway WebSocket is REST-ish. Either works; I picked AppSync because the chat use case is GraphQL-shaped.
6. **The "1 month late submission" rule** — I implemented this as S3 lifecycle. But the *application* also needs to know whether a submission is late, which is a piece of business logic in the `submission-api` Lambda.

---

## Step 8 — How the marker will read this

I structured the essay to match the question's structure:

- **Cloud-service category table** — directly answers Task 1's "which cloud services should be used and how they can be used for which purposes".
- **Requirements mapping table** — same pattern as Q1, Q2: every requirement maps to a named component.
- **End-to-end flows** — three concrete scenarios (upload, grade, chat) to make the architecture feel real.
- **Task 2 — login and roles** — has its own section with an OAuth flow diagram, JWT example, and an explicit RBAC explanation.
- **Diagram** — single picture; serverless-first is visually obvious because there are no VMs anywhere.

The "rapid development" message is woven through; the cloud-categories table makes it concrete; the trade-offs section shows I know the limits.
