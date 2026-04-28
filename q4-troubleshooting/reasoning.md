# Q4 — How I Thought Through This (Step-by-Step)

The polished answer is in `essay.md`. This file captures the thinking.

---

## Step 1 — What is the question actually asking?

Q4 is different from Q1–Q3. It's not an architecture question; it's a **diagnostics** question. Two tasks:

1. **Method** to find the cause of "cloud apps cannot reach on-prem DB".
2. **List** possible causes; pick one and design a fix.

So the deliverable isn't a system; it's a **process** plus a worked example.

---

## Step 2 — Pick the diagnostic mental model

I had two candidate frameworks:

| Framework | What it gives | When it's strong |
|---|---|---|
| **OSI bottom-up** | Layer-by-layer checklist (L1 → L7) | Familiar to the course; explicitly in the cheat sheet |
| **"Walk the packet"** | Trace the actual path from source to destination | Better for hybrid topologies with many hops |

I went with **both**, because they cover different blind spots:

- "Walk the packet" tells you *where* on the path to look.
- "OSI bottom-up" tells you *what* to check at that location.

If I only used OSI, I'd ask "is L3 OK?" without knowing which device's L3 to check. If I only walked the packet, I might miss the layer (e.g., debug TLS while the tunnel is down). Together they're stronger than either alone.

---

## Step 3 — List the hops

For an AWS hybrid path, the hops are:

```
Cloud App (Lambda/EC2)
  → VPC Subnet + ENI
    → Security Group + NACL
      → VPC Route Table
        → Virtual Private Gateway (VGW)
          → IPsec tunnel over the internet
            → On-prem firewall + VPN endpoint
              → On-prem switch / VLAN
                → DB server (listener, auth, app)
```

Eight hops. I labelled each hop in the diagram with a **"Check N"** callout that says exactly what to inspect there. This is what makes the diagram useful as a runbook in a video presentation.

---

## Step 4 — The "first three commands" trick

A senior engineer (or anyone who's debugged this before) doesn't run the full checklist top-to-bottom. They start with the highest-yield checks. I forced myself to write down what those are:

1. **`aws ec2 describe-vpn-connections`** — answers "is the tunnel up?" in 10 seconds.
2. **VPC Reachability Analyzer** — answers "where is the packet getting blocked?" in another 30 seconds. AWS does the path analysis for you.
3. **`telnet on-prem-db-ip 5432`** — answers "is it network or app?" If TCP handshake succeeds, it's L5–L7 (TLS, auth, DB). If it fails, it's L3–L4 (route, SG, FW).

These three pin down the failing layer in under five minutes. Everything else in the checklist exists to drill down once you know which layer.

I wanted this in the essay because it shows the marker that I understand *prioritising* the diagnostic, not just reciting it.

---

## Step 5 — Listing possible causes (Task 2 part 1)

I brainstormed by going hop-by-hop and asking "what could go wrong here?" That gave me the 13 causes in the essay. I deliberately ordered them roughly by frequency in real life:

- VPN tunnel down (1) — single most common.
- Route table missing (2) — common right after initial deployment.
- SG / NACL / on-prem FW (3-5) — common, often forgotten.
- Asymmetric routing (6) — subtle, hard to find.
- CIDR overlap (7) — subtle, design-time mistake.
- DNS / app / DB-level issues (8-11) — happens when the network is fine.
- Lambda not in VPC (12) — easy to miss, the symptom looks like network failure.
- MTU / fragmentation (13) — rare but classic.

I included #12 deliberately because as a software engineer I find it the easiest one to get wrong — Lambda has a "VPC config" toggle that's off by default, and if it's off the function reaches the internet, never the VGW. Symptom: looks identical to a tunnel-down problem.

---

## Step 6 — Picking one to fix (Task 2 part 2)

The question says "select **a** problem and design a solution". I had three candidates:

| Candidate | Pros | Cons |
|---|---|---|
| **VPN tunnel down** | Most common; tests IPsec knowledge; AWS provides excellent tooling | Slightly network-specialist for a SWE |
| Security Group missing rule | Most common AWS-only mistake; trivially in my comfort zone | Boring fix — one console click |
| CIDR overlap | Most subtle, most educational | Hard to fix mid-deployment without re-IP'ing |

I picked **VPN tunnel down**. Reasoning:

- It exercises the most layers (L3 networking, IPsec configuration, AWS-side and on-prem-side coordination).
- The fix loop demonstrates real diagnostic tools (`describe-vpn-connections`, Reachability Analyzer, CloudWatch metrics) that the marker can verify I understand.
- It's by far the most common real-world cause.
- The fix has steps the marker can grade — it's not just "click here".

**What I'm less confident on**: the IKE/IPsec proposal vocabulary (DH groups, DPD, AES-256-GCM vs CBC, etc.). I described the principle ("compare both sides line-by-line, find the mismatch, restart") without claiming to know every parameter from memory. That's deliberate — the master's-level expectation is that I know the *shape* of the problem, not that I'm a Cisco-certified network engineer.

---

## Step 7 — The closing detail: prevent it next time

Adding the CloudWatch alarm on `TunnelState = 0` at the end of the fix is a small thing but signals senior-engineer thinking: *don't just fix it, make sure you find out next time before the deploy fails.*

This is the kind of follow-through that earns the "detailed and convincing" wording in the marking guidance.

---

## Step 8 — Things I'd dig deeper on

Honest list:

1. **IPsec proposal vocabulary** — I'd revise the exact AES/DH/lifetime parameters AWS recommends.
2. **BGP vs static routing on the VPN** — for HA you'd use BGP; for static you don't propagate routes automatically. I hand-waved this.
3. **CIDR overlap mitigation patterns** — the fix involves either re-IP'ing or using a private NAT in front; both deserve their own essay.
4. **MTU / fragmentation on IPsec** — I mentioned it but didn't dig in. The classic symptom is "small packets work, big ones don't"; the fix is TCP MSS clamping. Worth knowing.
5. **DNS in hybrid setups** — Route 53 Resolver outbound endpoints and conditional forwarders. I cited "DNS resolution failure" as a possible cause but didn't expand.

---

## Step 9 — How the marker will read this

The structure of the answer is built to match the question's structure exactly:

- **Method to determine the cause** (Task 1) → walk-the-packet table + OSI bottom-up table + the explicit "devices, modules, networks, services, software" enumeration.
- **List possible problems** (Task 2 first half) → 13 numbered causes, layer-tagged.
- **Pick one + design fix** (Task 2 second half) → "VPN tunnel is down" + nine concrete fix steps + a "why it failed in the first place" insight + an alarm to prevent recurrence.
- **Diagram** → both halves of the answer in one picture: hops with checks on top, OSI + problem list on the bottom.
