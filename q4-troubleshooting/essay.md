# Question 4 — Troubleshooting Cloud Services

**Course:** Network and Cloud Essentials — Final Exam
**Author:** Nyi Htut Zaw
**Date:** 28 April 2026

---

## 1. Scenario Recap

The university student-portal platform cannot be deployed because the **cloud apps cannot reach the on-prem database**. The development team needs to find the root cause and fix it.

The exam tasks:

- **Task 1**: Design a method to determine the cause. Which devices, modules, networks, services, and software should be investigated, and how?
- **Task 2**: List possible problems. Pick one and design a fix.

---

## 2. Diagram

![Cloud-to-Onprem DB Troubleshooting](diagram.png)

The diagram has three regions:

- **Top half** — the normal hybrid path with a *Check N* callout at every hop.
- **Bottom-left** — OSI bottom-up diagnostic order with the tools to use at each layer.
- **Bottom-right** — the list of possible causes plus the one we picked to fix.

---

## 3. Task 1 — Diagnostic Method: "Walk the Packet" + OSI Bottom-Up

Think of troubleshooting like finding where a letter got lost in the mail. You don't search everywhere at once; you trace the route the letter took and check each step.

We use two complementary approaches together:

1. **Walk the packet** — imagine following a data packet from the cloud app to the on-prem database. At each stop along the way, ask: *can the packet still get through here?* This tells us which section of the path is broken.

2. **OSI bottom-up** — once we've identified a broken section, we check it from the lowest layer (physical cables) up to the highest layer (applications). This matters because if the cables are disconnected, there's no point checking the application settings. Start with the basics first.

This is the method taught in the course, and it works well for hybrid AWS scenarios where you have both cloud and on-premises systems talking to each other.

### 3.1 The five hops along the path

**Hop 1: Cloud app (EC2 or Lambda server).** The application needs to know: what's the database hostname? What port? What username and password? Check the connection string in the app's code or config file. When there's an error, the logs will tell you if it's a network problem ("connection timeout" = network is slow or down) or a credential problem ("auth failed" = wrong password or username).

**Hop 2: Cloud network (VPC security and routing).** AWS has three layers of security and routing: the subnet (which network segment is the app in?), the Security Group (like a firewall that says "traffic to port 5432 is allowed out"), and the Route Table (tells packets "to reach the on-prem database, send it through the VPN tunnel"). Check that all three are correctly configured. AWS tools: use the VPC console to see the settings; use **VPC Reachability Analyzer** to test if packets can travel from the app to the database.

**Hop 3: VPN tunnel (the encrypted connection between cloud and on-premises).** The tunnel is like a secure phone line between AWS and your office. Both sides need to agree on how to encrypt/decrypt the call, and both sides need to have the same secret key. Check that the tunnel shows as "UP" on both sides, and that the secret key and encryption settings match. AWS tools: `aws ec2 describe-vpn-connections` to check the tunnel status; on your office equipment, there's usually a command like `show crypto` to verify the tunnel is established.

**Hop 4: On-prem network (your office firewall, switches, and routing).** Your office firewall is like a bouncer at a club — it decides which traffic is allowed in and which is blocked. Make sure the firewall allows traffic from the cloud (on port 5432 for the database). Also make sure your office network knows how to route data back to AWS (no "return path mystery" where the response goes somewhere else). Tools: check the firewall rules in your office IT console; check the office router's routing table; verify that the database server is on the right network segment.

**Hop 5: Database server itself.** The database server must be listening for connections on port 5432 (for PostgreSQL) or 3306 (for MySQL). The database also has its own access control — it needs to allow connections from the cloud app's IP address, and the username/password must be correct. Tools: log in to the database server and run `netstat -lntp` to see what ports are listening; check the database logs; try connecting from the cloud using a test command like `psql` or `mysql` to see what error the database gives.

### 3.2 OSI bottom-up checklist

When something doesn't work, start from the bottom and work your way up. If the cable is broken, there's no point checking the database settings.

**Level 1 — Physical (cables and wires).** Check that cables are plugged in, lights are on, and the on-prem equipment is powered on. AWS handles this for us on their side, so we assume it's fine. This is literally "look at the hardware."

**Level 2 — Local network (switches and MAC addresses).** On-prem: the network switch needs to know which cable the database server is plugged into. Check that the database server is plugged into the right switch port on the right network segment (VLAN). Tools: look at the switch console; run `arp -a` to see if the database IP is known.

**Level 3 — Routing (getting packets across the internet).** Both cloud and on-prem need to know "how do I send packets to the other side?" In the cloud, the route table says "on-prem traffic goes through the VPN." On-prem, the router needs a similar entry saying "cloud traffic goes out this way." Also check: is the VPN tunnel UP? Do the cloud and on-prem networks use different IP ranges (if they overlap, routing gets confused). Tools: **VPC Reachability Analyzer** (AWS tool to test if traffic can flow); run `traceroute` (shows the path a packet takes); `aws ec2 describe-vpn-connections` (checks if the VPN tunnel is UP).

**Level 4 — Firewall rules.** The cloud Security Group (firewall) must allow traffic OUT to the database port (5432). On-prem, the office firewall must allow traffic IN from the cloud on the same port. A common mistake: the outbound rule says "allow port 5432 out" but the return traffic (responses from the database) is blocked. Tools: `telnet on-prem-db-ip 5432` from the cloud (tests if the connection handshake works); **VPC Flow Logs** (shows if packets are being rejected).

**Level 5/6 — Encrypted connection (TLS).** If the database requires a secure encrypted connection, check that the encryption certificate is valid and not expired. Tools: `openssl s_client -connect db:5432` (shows the certificate details).

**Level 7 — Application (database itself).** The database must be listening for connections, and it must allow the cloud app's IP address to connect. Check that the database process is actually running, the username/password are correct, and the hostname resolves to the right IP. Tools: log in to the database server and use the database client (`psql` for PostgreSQL, `mysql` for MySQL); check the database logs for errors.

### 3.3 What to check: devices, modules, networks, services, and tools

To be thorough, here's what you need to investigate:

- **Devices (physical things)**: the cloud server running the app (EC2 or Lambda), the VPN gateway in AWS, the office firewall, the office router, the network switch, and the database server itself.

- **Configuration modules (settings)**: the VPN tunnel encryption settings (both sides), the routing rules (how packets should flow), the firewall rules (what traffic is allowed).

- **Networks (the paths between things)**: the cloud network IP range (CIDR), the on-prem network IP range, and the internet connection path between AWS and your office.

- **AWS services and tools**: VPC Reachability Analyzer (tests if packets can flow), CloudWatch (shows if the VPN tunnel is up), VPC Flow Logs (shows if packets are being rejected), Route 53 (DNS — so hostnames resolve to IP addresses).

- **Software tools you run**: `ping` (test if a server is reachable), `traceroute` (see the path), `telnet` (test if you can connect to a port), `psql`/`mysql` (database clients to test the actual connection), `aws ec2 describe-vpn-connections` (AWS CLI command to check tunnel status).

### 3.4 Start with the quickest checks

Instead of checking everything at once, start with the three things most likely to be the problem:

1. **`aws ec2 describe-vpn-connections`** — is the VPN tunnel even UP? This is the #1 cause of "cloud can't reach on-prem" problems. Takes 10 seconds to check. If the tunnel is down, everything else is pointless.

2. **VPC Reachability Analyzer** (AWS tool) — tell it "I want to send traffic from my cloud app to the on-prem database" and it will tell you exactly which step is failing. Very useful because it narrows down from 5 hops to 1.

3. **`telnet on-prem-db-ip 5432`** from a cloud instance — try to open a connection to the database. If it works, you know the network is OK and the problem is probably credentials or the database config. If it fails, you know the network itself is broken and you need to go back and check the firewall rules and routing.

These three quick checks usually tell you which layer is broken, and then you can focus your investigation there.

---

## 4. Task 2 — Possible Problems

When cloud can't reach on-prem, here are the most common causes (listed by likelihood):

1. **VPN tunnel is DOWN** — This is #1 because it's the most common. The encrypted connection between AWS and your office isn't established. Solution: check that the secret key and encryption settings match on both sides.

2. **Route table missing the on-prem route** — The cloud route table doesn't have an entry that says "traffic to the office network should go through the VPN tunnel." Solution: add the missing route.

3. **Cloud security group blocks outgoing traffic** — The EC2/Lambda's firewall rule doesn't allow traffic out on port 5432 (or whatever port the database uses). Solution: add an outgoing rule that allows this.

4. **Return traffic is blocked** — Traffic from cloud to office gets through, but the office's response back to the cloud is blocked by the office firewall. Solution: make sure the office firewall allows return traffic on the same port.

5. **Office firewall blocks incoming traffic** — The office firewall rule doesn't allow traffic from the AWS cloud network. Solution: add an inbound rule allowing the cloud's IP range.

6. **Packet goes out but never comes back** — Sometimes a packet reaches the destination but the return path is different, so the response never makes it back. This is rare but annoying to troubleshoot.

7. **IP ranges overlap** — The cloud network and office network accidentally use the same IP range (e.g., both use 10.0.0.0/16). The router doesn't know which one you mean. Solution: change one of the IP ranges.

8. **DNS can't find the database hostname** — The cloud app says "connect to database.company.com" but that hostname doesn't resolve to an IP address in the cloud. Solution: configure the cloud to ask the office's DNS server for hostname lookups.

9. **Database isn't listening** — The database service isn't running, or it's listening on the wrong port, or it's only listening to local connections (127.0.0.1). Solution: start the database or fix the port/settings.

10. **Database won't let the cloud connect** — The database has its own access control (like pg_hba.conf in PostgreSQL). It doesn't recognize the cloud app's IP as a valid client. Solution: add the cloud's IP to the database access list.

11. **Certificate problem** — If the database requires an encrypted connection, the certificate might be expired or invalid. Solution: renew the certificate or disable certificate checking if it's just for testing.

12. **Lambda runs outside the VPC** — The Lambda function isn't configured to run inside the cloud's network, so it goes out through the public internet instead of the VPN. Solution: configure the Lambda to run in a VPC.

13. **Network packet size problem** — The VPN tunnel has a size limit on packets. If you try to send a packet larger than this limit, it gets broken into pieces, and sometimes those pieces get lost. Solution: adjust the packet size or change the tunnel settings.

---

## 5. Selected Problem and Fix: VPN Tunnel Is Down

I'm choosing **Problem #1 — VPN tunnel is down** because (1) it's the most common cause of "cloud can't reach on-prem," and (2) it shows how to diagnose and fix a real problem from start to finish.

### 5.1 How you know it's this problem

Go to the AWS console and check:

1. **AWS console → VPC → Site-to-Site VPN Connections** — look for the status. If it says **DOWN**, that's the problem right there.
2. **AWS CLI** — run `aws ec2 describe-vpn-connections` and look for status messages like *"IKE failed"* or *"Phase 2 negotiation failed"*.
3. **VPC Reachability Analyzer** (AWS tool) — try to send traffic from the cloud app to the on-prem database. If it says "Not reachable" and mentions "no active VPN tunnel," that confirms it.
4. **On-prem firewall/router** — log in and look for any error messages about the VPN connection. Different equipment has different commands, but usually something like `show crypto` or `show vpn status`.

### 5.2 How to fix it

The VPN tunnel is like a phone call between AWS and your office. Both sides need to agree on how to encrypt the call and use the same secret key. Here's how to fix it:

1. **Get the AWS config file** — go to AWS console, find your VPN connection, and download the configuration. This shows you what AWS expects.

2. **Compare with your office config** — have someone log in to the on-prem firewall/router and check the VPN settings. Look for these things (they must match exactly on both sides):
   - **Secret key** — did someone make a typo when copying it?
   - **Encryption method** — both sides should use AES-256 or whatever was agreed.
   - **Network ranges** — cloud side says "office network is 10.0.0.0/16", office side says "AWS network is 10.1.0.0/16". They should match.

3. **Fix whichever side is wrong** — update the config on that side. On the office equipment, you might need to clear the old tunnel and let it re-establish (different equipment has different commands, like `clear crypto` on Cisco).

4. **Check the AWS console** — watch the tunnel status. It should change from DOWN to UP within 1-2 minutes.

5. **Test from the cloud** — try `telnet on-prem-db-ip 5432` from the cloud to see if you can connect to the database now.

6. **Test from the app** — actually run the application and see if it can read from the database now. If it still fails, the network is working but something else is wrong (credentials, database permissions, etc.).

7. **Prevent next time** — set up an alert in CloudWatch so that if the VPN tunnel goes down again, someone gets notified immediately instead of finding out when the app breaks.

### 5.3 Why the tunnel failed (likely reasons)

Usually a tunnel works fine for months, then suddenly breaks. Why?

- **Someone changed the firewall settings** on the office side without updating AWS, or vice versa.
- **The secret key got corrupted** during a backup or update.
- **AWS updated their VPN endpoint** and the on-prem side wasn't notified.
- **There was a typo** when setting up the config in the first place, and it only now got noticed.

To find out which, check the change logs: AWS CloudTrail shows what changed on the AWS side, and your office IT should have a change log showing what changed on the office side.

---

## 6. Summary

When the cloud app can't reach the on-prem database, you troubleshoot it in two steps:

**Step 1: Walk the packet** — trace the data from the cloud app all the way to the database server, checking at each stop ("is the firewall allowing this? is the router configured right? is the database listening?").

**Step 2: Check from the bottom up** — start with the physical cables, then check the network routing, then the firewalls, then the application. If the cables are broken, there's no point checking the database settings.

**Three quick checks** that usually pinpoint the problem: (1) Is the VPN tunnel UP? (2) Does AWS Reachability Analyzer say the database is reachable? (3) Can you connect to the database port with telnet?

**The most common problem** is a VPN tunnel that's DOWN because the cloud and office configurations don't match. The fix: compare the two sides' settings, find what's different, update it, and restart the tunnel.

This approach is learned from the course's troubleshooting method, and it works well for any cloud-to-on-prem connectivity problem.

---

## References

1. AWS — *VPC Reachability Analyzer*. https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html
2. AWS — *Site-to-Site VPN troubleshooting*. https://docs.aws.amazon.com/vpn/latest/s2svpn/Troubleshooting.html
3. AWS — *VPC Flow Logs*. https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html
4. AWS — *CloudWatch metrics for VPN*. https://docs.aws.amazon.com/vpn/latest/s2svpn/monitoring-cloudwatch-vpn.html
5. ISO/IEC 7498-1 — OSI reference model.
6. Course cheat sheet — OSI bottom-up troubleshooting (lecture material).
