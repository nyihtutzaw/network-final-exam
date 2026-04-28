# Q1 — How I Thought Through This (Step-by-Step)

This is for me. The polished answer is in `essay.md`; this file is the working-out.

I'm a senior software engineer learning networking and cloud at master's level — comfortable with code, APIs, and auth, but treating the networking-heavy parts as new ground. So I'll lean on what I know, pick the most obvious solid option (not the most clever one), and accept reasonable trade-offs.

---

## Step 1 — Read the question and underline what's required

Before drawing anything I made a checklist. Marker said "full marks for detailed and convincing supportive explanation" — so the risk is missing one requirement.

| # | What the question says | What it means in practice |
|---|---|---|
| 1.1 | Secure connection between cloud and campus | Encrypted tunnel → **VPN** |
| 1.2 | Private access to the database | DB not on the public internet → cloud apps reach it only via the tunnel |
| 1.3 | Public access only to web front end | Internet sees only the web tier → firewall + load balancer |
| 1.4 | Protection from credential theft | **MFA + SSO**, no SSH keys lying around |
| 2 | Separate user PCs from app/DB servers (on-prem **and** cloud) | Network segmentation in **both** zones |
| 3.1 | Prevent ransomware from spreading | Stop lateral movement → segmentation + endpoint protection |
| 3.2 | IR investigation + automated actions | Logs in one place + automatic response scripts |
| 4 | Backup to cloud + physical site, one super hard to delete | Multiple copies, one **immutable**, one **offline** |
| 5 | Patient data must be in Thailand | DB on-prem; AWS region = **Bangkok** |

Every component I added later had to point to one of these rows. If it didn't, I dropped it.

---

## Step 2 — Pick AWS region first (it's the only legal constraint)

Requirement 5 (patient data in Thailand) is non-negotiable. So:

- Patient DB stays on-prem (the question says so).
- All AWS resources run in **`ap-southeast-7`** (Bangkok). AWS launched this region in 2025, so it's the obvious choice.

**Trade-off accepted**: a newer region might have fewer services available than older regions like `us-east-1`. For a hospital workload that's fine — I'm not using anything exotic.

---

## Step 3 — Sketch the three zones

Three big boxes:

1. **On-premises hospital** — workstations, app server, patient DB, backup server.
2. **AWS cloud** — public web tier, app tier, identity, monitoring, immutable backup.
3. **Off-site physical site** — air-gapped tape vault.

Why three and not two? Because requirement 4 *explicitly* asks for a physical secondary site, separate from cloud storage. If I left it out I'd lose marks.

---

## Step 4 — Connect them safely

Two questions to answer:

**A. How does internet traffic reach the web front end?**
- Only HTTPS gets in.
- Cloud side: WAF + Application Load Balancer in a public subnet. WAF blocks SQL-injection and OWASP attacks.
- On-prem side: public access now via AWS CloudFront (no on-prem DMZ needed).

**B. How do cloud apps reach the on-prem DB?**
- Not over the public internet.
- Two reasonable AWS options:
  - **Site-to-Site VPN** — cheaper, slower, easy.
  - **Direct Connect** — dedicated link, faster, expensive, weeks to set up.
- For a small hospital, VPN is enough. **Trade-off accepted**: lower throughput and slightly higher latency, but much simpler. If traffic ever grows, swap in Direct Connect.

---

## Step 5 — Segmentation (the heart of the ransomware fix)

The original attack succeeded because everything was on one flat network — once one PC was infected, ransomware reached the file server easily. The fix is to split the network so even if one part is hit, the rest survives.

**On-prem** — three VLANs with a firewall between them:
- **Workstation VLAN** — staff PCs. Cannot reach DB or backup VLANs directly.
- **Server VLAN** — application server and patient DB.
- **Backup VLAN** — backup server only. Outbound only to cloud immutable storage and to tape.

**Cloud** — three subnets:
- Public (load balancer only)
- Private app (servers, multi-AZ)
- Management (admin access only)

This is the single most important control. Even if a PC is compromised, ransomware can't reach the DB or rewrite the backups.

**What I'm less sure about**: the exact firewall rules between VLANs. I know the principle (least privilege, deny by default). A real network engineer would tune things like ICMP, DHCP relay, and AD ports more carefully. For the exam I describe the principle, not actual ACL rules.

---

## Step 6 — Identity (req 1.4)

Credential theft is what almost always lets ransomware in. The fix is identity-first:

- **AWS IAM Identity Center** as the single login point. SSO with MFA.
- **AWS Systems Manager Session Manager** for admin shell access. No SSH keys, no inbound port 22 on any server. Every admin session is logged.

This was new to me. The point of Session Manager is that there's nothing for an attacker to credential-stuff against — there's no exposed port to attack.

---

## Step 7 — Detection and automated response (req 3.2)

Two pieces: **how do we know an attack is happening**, and **what happens automatically when it does**.

**Detection — multiple signals, not just one.** I added:
- **Endpoint protection** on every workstation and server — catches mass-encryption behaviour.
- **Trap files** ("honeyfiles") — fake docs no real user touches; first one accessed = strong signal.
- **AWS GuardDuty** for cloud threat detection.
- **CloudTrail + VPC Flow Logs** for the audit trail.

All this aggregates in **AWS Security Hub** for one dashboard.

**Automated response.** No real SOAR product; I'll build the equivalent with **AWS EventBridge → AWS Lambda**:
1. Cut the affected machine off the network.
2. Disable the user's login.
3. Snapshot disks for forensics.
4. Open a ticket.

Auto-containment in single-digit minutes — humans can't react that fast. Humans take over for analysis and recovery later.

---

## Step 8 — Backups (the 3-2-1-1-0 rule)

The course material introduced **3-2-1-1-0**:

- **3** copies of data
- **2** media types (disk + tape)
- **1** off-site copy
- **1** immutable / offline copy
- **0** errors after restore tests

For requirement 4 ("one backup must be super hard to access/delete") I want **two** hard-to-delete copies for safety:

- **Cloud**: **Amazon S3 with Object Lock**. Write-once for the retention period. Even an attacker with admin credentials cannot delete it before retention ends.
- **Physical**: physical tape (LTO), written via **AWS Storage Gateway**, then taken off-site. Tape is offline — ransomware cannot reach it over the network.

**Trade-off accepted**: tape is slow to restore, so it's the last-resort copy, not the first one we'd reach for. That's why we have the cloud immutable copy too.

---

## Step 9 — Incident Response Plan

Three things to deliver: framework, detection, action timeline.

**Framework choice**: **NIST SP 800-61 Rev. 2** vs **SANS PICERL**. Same idea (prepare → detect → contain → eradicate → recover → learn). I picked NIST because:
1. It's what the lecture slides used.
2. It's commonly cited by Thai regulators, which fits the PDPA angle.

**Detection table**: I listed the typical signals. The point: don't depend on any single indicator.

**Timeline**: T+0 (alert) to T+1 week (post-mortem). Important things to call out in the video:
- Auto-containment in **single-digit minutes** — that's what limits the damage.
- Humans take over from T+5 onwards.
- **PDPA 72-hour breach notification** — Thai law, must be in the timeline.

---

## Step 10 — What I'd dig deeper on if this were real

Honest list of gaps:

1. **Firewall rules between VLANs** — I described the principle; a real implementation needs careful tuning around AD, DNS, monitoring traffic.
2. **Endpoint protection product choice** — name brands are a procurement decision.
3. **Detection rule tuning** — false-positive management is a real-world pain I'm hand-waving past.
4. **Tape rotation logistics** — chain of custody, vault security, restore-test cadence.
5. **Network diagrams** — using Cisco shapes from drawio because the lectures used them; a real network engineer's diagram would be tidier.

---

## How this maps to what the marker is looking for

> Full marks are given to a solution with detailed and convincing supportive explanation.

The structure of the essay matches this:

- **Mapping table** — proves every requirement is addressed.
- **Per-component justification** — proves I know **why** each piece is there, not just that it's there.
- **Trade-offs explicit** — shows I understand the alternatives, even when I don't pick them.
- **IR plan with timeline** — answers Task 2 directly with concrete numbers (T+0, T+5, T+72hr).
- **Diagram** — single picture the marker can scan and tick off against the component table.
