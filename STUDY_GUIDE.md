# Study & Video-Explanation Guide — Final Exam

Read this **first** before you start studying or recording videos. It's the map.

---

## 1. What's in each question folder

```
q1-ransomware-hospital/
q2-fintech-api/
q3-student-submission/
q4-troubleshooting/
    ├── essay.md       ← the polished answer to submit / show
    ├── reasoning.md   ← how I thought through it (private — for your study)
    ├── diagram.drawio ← editable architecture diagram
    └── diagram.png    ← exported image, embedded in the essay
```

Two files matter for the exam:

- **`essay.md`** — this is the deliverable. The marker reads this. It is **declarative**: states the design, justifies each component, answers each task directly.
- **`diagram.png`** — the picture you show on screen during the video.

Two files are for **you**:

- **`reasoning.md`** — the step-by-step thinking. Read this *while* you study so you understand *why* each choice was made, not just *what* the choice is. The marker will not see this; you internalise it.
- **`diagram.drawio`** — open this in drawio.app if you want to tweak labels, colours, or layout before recording.

---

## 2. How to study (suggested order, ~3 hours total)

### Phase 1 — Get the shape (30 min)

1. Skim the **PDF question** for each of Q1–Q4 again. Re-read the requirements bullet list and the Tasks list.
2. Open each `diagram.png` and look at the picture. Don't read anything yet — just absorb the shape.
3. After looking at all four diagrams, you should be able to describe the four scenarios in one sentence each. Try it out loud.

### Phase 2 — Walk through one question deeply (45 min per question)

Pick the question you'll record first. Then, for that question:

1. **Read the essay's section §1 (Scenario)** to refresh what's being asked.
2. **Look at the diagram** and identify the three zones (cloud / on-prem / off-site) before reading anything else.
3. **Read the requirements-mapping table** in the essay — this is the spine. Every requirement has a pointer to a component.
4. **Read the per-component justifications** — these are what you will say *out loud* in the video.
5. **Then read the reasoning.md** — this fills in *why* I made each choice. After reading it you'll be able to defend trade-offs if the marker asks.
6. **Look at the diagram again** — by now every box should mean something specific to you.

Repeat this for each question.

### Phase 3 — Cross-cutting patterns (30 min)

Notice that the same patterns appear in multiple questions:

| Pattern | Where it appears |
|---|---|
| **Hybrid VPN + private DB** | Q1, Q2, Q3 (and is the failure mode in Q4) |
| **Cognito + JWT + SSO** | Q2 (mobile), Q3 (students/teachers) |
| **3-2-1-1-0 backup** | Q1, Q2, Q3 |
| **AWS Backup + S3 Object Lock + Storage Gateway tape** | Q1, Q2, Q3 |
| **GuardDuty + Security Hub + EventBridge → Lambda for SOAR** | Q1, Q2, Q3 |
| **Multi-AZ for HA** | All four |
| **OSI bottom-up troubleshooting** | Q4 directly; underlies the design choices in Q1–Q3 |

If the marker asks "why did you use this same pattern again?", the answer is "because it's the standard solution for this class of problem, and reusing it across the platform is the right call". This is a *strength* of the answer, not a weakness.

### Phase 4 — Memorise the key numbers and terms (15 min)

Things you should be able to say without looking:

- **3-2-1-1-0** — backup rule (3 copies, 2 media, 1 off-site, 1 immutable, 0 errors after restore).
- **NIST SP 800-61** — incident response framework.
- **PDPA** — Thai Personal Data Protection Act; **72 hr** breach notification window.
- **`ap-southeast-7`** — AWS Bangkok region.
- **OAuth 2.0 / OIDC + JWT** — standard auth pattern.
- **OSI L1 → L7** — the layers, in order, for Q4.
- **The first three diagnostic commands for Q4** — `aws ec2 describe-vpn-connections`, VPC Reachability Analyzer, `telnet`.

---

## 3. How to record the video (per the exam instructions)

The exam says:

> Video recording of your explanation for each question must be submitted via a private YouTube video link. The video should present your write-up, diagram, table, references, and have your face while talking in a small box.

So each question needs a separate video. **Plan ~8–12 minutes per question.** Longer is not better — focused is better.

### 3.1 Video structure (use the same template for all four)

```
0:00–0:30   Hook — read the scenario in one sentence and state what you're solving.
0:30–1:30   Show the diagram. Walk the three zones (cloud / on-prem / off-site).
1:30–4:00   Walk the requirements-mapping table. For each row: "Requirement X
            → component Y → here it is on the diagram."
4:00–7:00   Component-by-component justification. Pick the 4–6 most interesting
            components; don't read every paragraph. For each: "I picked X because
            of Y. Trade-off accepted: Z."
7:00–9:00   Answer the specific Tasks (Task 2, Task 3 etc. depending on question).
            Use the flow diagrams in the essay.
9:00–10:00  Summary + references. Close with one sentence on what makes this
            design hold up under stress.
```

### 3.2 What to show on screen at each stage

- **Hook**: the question PDF page (zoomed to the scenario).
- **Diagram walk-through**: full diagram, then zoom into each zone.
- **Requirements table**: the table in the essay.
- **Component justifications**: the diagram with the relevant component highlighted; cut to the essay text only when it helps.
- **Tasks**: the flow diagram in the essay (e.g., the OAuth/JWT flow in Q2 §5, the OSI table in Q4).
- **Summary**: the diagram one last time.

Avoid reading the essay verbatim. Skim it; speak in your own words. The marker is checking that you *understand* it, not that you can read it.

### 3.3 Per-question talking points (the things you must say)

**Q1 — Hospital ransomware**
- "Old design lost data because the file server and its backups were on the same flat network."
- "New design fixes three things: microsegmentation, immutable backup, and SIEM-driven auto-containment."
- "3-2-1-1-0 backup rule: I have **two** super-secure copies — S3 Object Lock and air-gapped tape."
- "NIST SP 800-61 framework. PDPA breach notification within 72 hours."
- "Auto-containment in single-digit minutes — humans can't react that fast."

**Q2 — Fintech API**
- "Lambda is the obvious choice for payday spikes — it auto-scales without warning."
- "One login, all endpoints: Cognito issues a JWT, every API Gateway endpoint accepts the same JWT."
- "Geo-fraud uses the user's Google Maps Timeline via OAuth — but I have a fallback for users who don't consent (IP geolocation + device GPS, with lower transaction limits)."
- "Eavesdropping → TLS 1.3. SQL injection → AWS WAF managed rules + parameterised queries. DDoS → Shield Advanced + CloudFront."

**Q3 — Student submission**
- "The whole answer is shaped by 'minimal upfront hardware'. So: serverless-first."
- "Walk through the cloud-service categories table: SaaS, PaaS, FaaS — that table is the answer to Task 1."
- "File flow: pre-signed S3 PUT → S3 PUT event → virus scan Lambda → if clean, plagiarism Lambda → notification Lambda → SES email. Every step is event-driven."
- "Roles via Cognito groups in the JWT. API Gateway and Lambdas check `cognito:groups` for RBAC."
- "Student profile stays on-prem; cloud writes grades back via the on-prem grade API over the VPN."

**Q4 — Troubleshooting**
- "Two frameworks: walk the packet + OSI bottom-up. Used together they cover blind spots."
- "First three commands solve 80% of these in five minutes: `describe-vpn-connections`, Reachability Analyzer, `telnet`."
- "13 possible causes; I picked VPN tunnel down because it's the most common and tests the most knowledge."
- "Fix is to align both sides' IPsec config — most common cause is a typo'd PSK or a mismatched proposal."
- "Always end the fix by adding a CloudWatch alarm so you find out before the next deploy fails."

### 3.4 What to do if the marker asks tough questions

The reasoning.md files prepare you for this. Common pushback and how to handle it:

| If they ask | You say |
|---|---|
| "Why AWS specifically?" | "I'm most familiar with AWS, the question doesn't mandate a cloud, and AWS has a Thai region (`ap-southeast-7`) that satisfies the data-residency requirement." |
| "Why not Direct Connect instead of VPN?" | "Direct Connect is faster and more reliable but costs more and takes weeks to provision. VPN is the right starting point. I'd upgrade if traffic justified it." |
| "Why Lambda over containers?" | "Lambda auto-scales without warning, fits the spiky load patterns in Q2/Q3, and means the team writes business logic instead of operating servers." |
| "Why three backup copies, not two?" | "Because the question explicitly asks for cloud + physical site, with one super-secure copy. Three copies (production + cloud immutable + air-gapped tape) is the minimum that satisfies that." |
| "Are you sure Compliance mode / 'last 3 months' / [specific detail]?" | "I'm not 100% sure on that exact parameter; the principle is X, and in a real implementation I'd verify against current AWS documentation before shipping." |

**It is OK to say "I'm not sure on that exact parameter" — it's better than guessing wrong. Confidence about the *principle* matters more than encyclopedic knowledge of every AWS parameter.**

---

## 4. Tooling reminders

You already have:
- **drawio.app** — to open `.drawio` files and tweak diagrams.
- **pandoc** — when you're ready, run `pandoc essay.md -o essay.docx` (or batch all four) to produce Word docs.

Nothing else to install.

---

## 5. Final checklist before you hit Record

For each question:

- [ ] Re-read the question PDF page, especially the **Tasks** list. Confirm each Task is explicitly answered in the essay.
- [ ] Read the essay's requirements-mapping table — every requirement has a component pointer.
- [ ] Look at the diagram on a big screen. Every box should make sense to you.
- [ ] Skim the reasoning.md so you know which trade-offs to call out.
- [ ] Write a 5-bullet talking-point list (use the per-question lists above as the starting template).
- [ ] Do one practice run through the video script, no recording. Aim for ≤ 12 minutes.
- [ ] Then record.

Good luck.
