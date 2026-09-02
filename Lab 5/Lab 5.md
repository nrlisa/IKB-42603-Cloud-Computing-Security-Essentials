# Lab 5: Monitoring, Logging & Incident Detection
Course: IKB42603 Cloud Computing Security Essentials

Topic: Centralised logging, tamper-proof logs, threat detection and incident response — Docker & LocalStack

Environment: Docker, LocalStack (fake AWS CloudWatch Logs), sha256sum, grep, awk

Name: Nurlisa Sofiya binti Mahadzir

## Lab Summary // Objective

This lab demonstrates monitoring, logging, and incident detection across centralised logging, tamper-proof logs, threat detection, and incident response:

- Application logs were generated and centralised into CloudWatch Logs (LocalStack), proving that scattered logs can be collected into one place for querying.
- Failed login attempts were queried and grouped by IP address, demonstrating how security-relevant activity is extracted from raw logs.
- A tamper-evident (hash-chained) log was built and verified — changing any line broke the chain, proving that log integrity can be cryptographically enforced.
- An incident was detected by correlating multiple events: repeated login failures followed by a success and a large data export from the same IP — a pattern no single log line would reveal.
- Incident response was executed: the attacker IP was contained with an iptables rule, evidence was collected with a hash, and a short incident report documented the timeline.

## Architecture Diagram

The following architecture models the monitoring, logging, and incident detection pipeline implemented across Sessions A and B.

```text
              Monitoring, Logging & Incident Detection (Lab 5)
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
     ┌────────┴────────┐   ┌───────┴───────┐   ┌────────┴────────┐
     │   Session A     │   │   Session B   │   │   Session B     │
     │  Log & Query    │   │ Tamper-Proof  │   │  Detect &       │
     │  Tasks 1–3      │   │ Tasks 4–5     │   │  Respond        │
     └────────┬────────┘   └───────┬───────┘   └────────┬────────┘
              │                     │                     │
              │                     │                     │
   ┌──────────┴──────────┐ ┌───────┴────────┐  ┌────────┴────────┐
   │                     │ │                │  │                 │
   │  auth.log           │ │  auth.chain    │  │  Correlation    │
   │  (raw events)       │ │  (hash-linked) │  │  Engine         │
   │                     │ │                │  │                 │
   │  ┌───────────────┐  │ │  ┌───────────┐ │  │  fails ≥ 3 ?   │
   │  │ LOGIN_OK      │  │ │  │ sha256    │ │  │  success ≥ 1 ? │
   │  │ LOGIN_FAIL    │  │ │  │ chain     │ │  │  export ≥ 1 ?  │
   │  │ EXPORT_DATA   │  │ │  │           │ │  │                 │
   │  └───────────────┘  │ │  │ any edit  │ │  │  ALERT if all  │
   │                     │ │  │ = broken  │ │  │  three match   │
   │  ┌───────────────┐  │ │  │   chain   │ │  │                 │
   │  │ CloudWatch    │  │ │  └───────────┘ │  └────────┬────────┘
   │  │ (LocalStack)  │  │ │                │           │
   │  │  centralised  │  │ │                │  ┌────────┴────────┐
   │  │  store        │  │ │                │  │  RESPONSE       │
   │  └───────────────┘  │ │                │  │                 │
   │                     │ │                │  │  CONTAIN:       │
   │  grep + awk         │ │                │  │  iptables DROP  │
   │  group by IP        │ │                │  │  203.0.113.9    │
   │                     │ │                │  │                 │
   └─────────────────────┘ └────────────────┘  │  COLLECT:       │
                                               │  evidence copy  │
                                               │  + sha256       │
                                               │                 │
                                               │  DOCUMENT:      │
                                               │  incident       │
                                               │  report         │
                                               └─────────────────┘
```

## Evidence Folder

All screenshots used for this report are stored in the `evidence lab 5` folder.

| **Evidence File** | **Purpose** |
|---|---|
| `lab5setup.png` | LocalStack setup — log group and log stream creation |
| `lab5task1.png` | Application log generation (auth.log with events) |
| `lab5task2.png` | Logs shipped to CloudWatch (put-log-events output) |
| `lab5task2_cloudwatch.png` | Centralised CloudWatch get-log-events read-back |
| `lab5task3.png` | Failed-login count grouped by IP |
| `lab5task4.png` | Hash-chained log and tamper-proof verification |
| `lab5task5.png` | Correlation ALERT output (brute-force → exfiltration) |
| `lab5task6.png` | Incident response — containment rule and evidence hash |
| `lab5verify.png` | Verification commands output |
| `lab5extra_soar.png` | SOAR-style automated IP blocking script (bonus) |

## Overview

This lab is split into two sessions:

- **Session A (Week 9):** Generate and centralise logs; query for failed logins (Tasks 1–3). Builds visibility.
- **Session B (Week 10):** Tamper-proof logs, incident detection and response (Tasks 4–6), then the incident report. Turns visibility into detection and response — the "prevention eventually fails" half of security.

Session A builds visibility. Session B turns that visibility into detection and response — the "prevention eventually fails" half of security.

**Security tip:** You cannot secure — or prove compliance for — what you cannot see. Logs are foundational to detection, forensics AND compliance evidence (Weeks 6, 10, 11).

---

## Setup — Start LocalStack

```bash
docker run -d --name localstack -p 4566:4566 \
  -e LOCALSTACK_AUTH_TOKEN=ls-xEbeGUwe-TOPE-lite-YOQu-ZoJI3462cb80 localstack/localstack
# f92aee4f709c440c827e0c4a8d1b337966c1aabb55bddd9db544dba9d5eaec4c

EP='--endpoint-url=http://localhost:4566'

aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

**Why:** LocalStack provides a fake AWS environment that behaves like the real thing. CloudWatch Logs is the centralised logging service — all logs from all services get shipped here so they can be queried from one place instead of SSH-ing into every host.

Evidence: <div align="left">
<img alt="LocalStack setup and log group creation" src="evidence lab 5/lab5setup.png">
</div>

---

## Session A (Week 9) — Logging & Centralisation

### Task 1 — Generate Application Logs

Create a small log of authentication events, including some failures (an attacker probing).

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF
cat auth.log
```

**Why:** This log models what a real application would write. Each line is a timestamped record of an event. Notice the pattern: four failed login attempts (brute force), then a success (the attacker guessed the password), then a large data export (exfiltration). No single line is alarming — the story emerges when you look at them together.

**Result:** The `auth.log` file contains 7 entries: one legitimate login, four failed attempts, one successful attack, and one data export.

Evidence: <div align="left">
<img alt="Application log generation" src="evidence lab 5/lab5task1.png">
</div>

---

### Task 2 — Centralise Logs (Ship to CloudWatch)

Send each line to the central log service — the cascading-collection idea from Week 6.

```bash
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null; TS=$((TS+1000));
done < auth.log

# Read them back from the central store
aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth \
  --query 'events[].message' --output text
```

**Why:** In production, logs from every service (web servers, databases, applications) get shipped to a central place (CloudWatch, Splunk, ELK). This lets you query across all sources without logging into each machine individually. The `get-log-events` read-back proves the logs arrived intact.

**Result:** All 7 log lines appear in the CloudWatch read-back, confirming successful centralisation.

```text
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5     2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9        2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9        2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9        2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9        2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9  2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
```

Evidence: <div align="left">
<img alt="Logs shipped to CloudWatch" src="evidence lab 5/lab5task2.png">
<img alt="CloudWatch centralised read-back" src="evidence lab 5/lab5task2_cloudwatch.png">
</div>

---

### Task 3 — Query for Security-Relevant Activity

```bash
# How many failed logins, and from which IP?
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

**Why:** Raw logs are useless without the ability to query them. This command groups failed login attempts by user and IP, counting occurrences. In a real SIEM, you would set up automated alerts for thresholds (e.g., "alert if >3 failures from one IP in 60 seconds").

**Result:** The query shows 4 failed login attempts, all from `user=admin ip=203.0.113.9`.

```text
      4 ip=203.0.113.9
```

**Log vs. Event distinction:**
- A **log** is a durable, timestamped record of something that happened (e.g., `LOGIN_FAIL user=admin ip=203.0.113.9`).
- An **event** is a trigger that fires in near real-time when a condition is met (e.g., "alert: 4 failures from 203.0.113.9").

Evidence: <div align="left">
<img alt="Failed-login count grouped by IP" src="evidence lab 5/lab5task3.png">
</div>

*End of Session A. Keep auth.log and the centralised read-back. Next week you will make these logs tamper-proof and use them to detect an incident.*

---

## Session B (Week 10) — Tamper-Proofing, Detection & Response

### Task 4 — Tamper-Proof (Hash-Chained) Logs

An attacker's first move is to edit the logs. Chain each line to the previous hash so any change breaks the chain.

```bash
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain
cat auth.chain
```

**Why:** A hash chain links each log entry to all previous entries. If an attacker modifies any line (e.g., changes `500MB` to `5MB` to hide the size of a data theft), the hash for that line changes, which cascades and changes every subsequent hash. This makes tampering detectable — you can prove that a log has not been altered since it was written.

```text
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5 | 82da89a49dc1ca7d23b8a59f98d7e557ab36ce0c2d0c6e106fabe76e1f0acf39
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9 | 790aef7176d6effe76d077831c071f8500204bf842e7fd8aeda1b67b2e271a97
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9 | 1e0b2e8aaf5143fb95070a8e57b009f058f0d37c257d19409b4131894d29a9a8
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9 | 7fb62c66ded511605e22c8db9c4f57c9360aa27309ce65024a3e5ea35e3b6e94
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9 | 143253b549a74b9626e910fbe54ca12cb5431a0a4c9c4f2189ff27a3e2a17e01
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9 | 4cbfab7fecb703cf21f5df81b47dbf3a727c94442b09b714ac4bfaa3584cc638
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB | ababa787b4bf524d9daddca8c48e4909fc105769a6f17574f42cefe8f81233cf
```

**Tamper verification:**

```bash
# Tamper: change the EXPORT size
sed 's/500MB/5MB/' auth.log > auth.tampered

# Recompute chain and compare final hash to auth.chain
ORIGINAL=$(tail -1 auth.chain | awk -F' \| ' '{print $2}')
PREV=0
while IFS= read -r line; do
    PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
done < auth.tampered
TAMPERED=$PREV

echo "Original final hash:  $ORIGINAL"
echo "Tampered final hash:  $TAMPERED"

if [ "$ORIGINAL" != "$TAMPERED" ]; then
    echo "RESULT: TAMPERING DETECTED - hashes are different"
else
    echo "RESULT: NO TAMPERING DETECTED - hashes are identical"
fi
```

**Proof — different hashes prove tampering was detected:**

```text
Original final hash:  ababa787b4bf524d9daddca8c48e4909fc105769a6f17574f42cefe8f81233cf
Tampered final hash:  72f1d53774a3a938fa7bd3a88f67894e5a64055a41ee7511eac53d7bd89d859b
RESULT: TAMPERING DETECTED - hashes are different
```

**Result:** The final hash of the tampered log (`72f1d5...`) differs from the original chain (`ababa7...`). Any edit — even a single character — produces a completely different hash, proving the chain detects alteration.

**Security tip:** Store the final hash (or forward the chain) to a separate, append-only location so an attacker who owns the app cannot also rewrite its audit trail (Week 6).

Evidence: <div align="left">
<img alt="Hash-chained log and tamper verification" src="evidence lab 5/lab5task4.png">
<img alt="Hash comparison proof — original vs tampered" src="evidence lab 5/lab5task4_compare hash.png">
</div>

---

### Task 5 — Detect the Incident (Correlation)

No single line was blocked, but together they tell a story. Detect the pattern: repeated failures, then a success, then a large export from the same IP.

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)

echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration';
fi
```

**Why:** This is what a SIEM (Security Information and Event Management) does: correlate events across sources into a single detection that no individual log would reveal. The brute force alone is suspicious. The success alone is normal. The export alone might be legitimate. But the combination — repeated failures from one IP, followed by a success from the same IP, followed by a large data export — tells a complete attack story.

**Result:** The script outputs:

```text
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

This confirms the incident was detected through correlation.

**Q: What is a SIEM?** A SIEM collects logs from multiple sources, correlates events across them, and generates alerts when patterns match known attack signatures. It turns raw logs into actionable security intelligence.

Evidence: <div align="left">
<img alt="Correlation ALERT output" src="evidence lab 5/lab5task5.png">
</div>

---

### Task 6 — Incident Response

Run the response lifecycle: contain, collect evidence, and document. Work quickly but preserve integrity.

```bash
# CONTAIN: block the attacker IP (model with an iptables rule)
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
  'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

```text
# (first run pulls alpine:latest)
# Status: Downloaded newer image for alpine:latest
target     prot opt source               destination
DROP       all  --  203.0.113.9          0.0.0.0/0
```

```bash
# COLLECT: make an immutable, timestamped evidence copy with its hash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

```text
0adc5d2ac06cbbdd366099bcc0540c4c0f76946e71b52e4c99322731696a203b *evidence_20260902.log
```

**Why:** Incident response has four steps, each with a specific goal:

| Step | Goal | What we did |
|---|---|---|
| **Detect** | Identify that an incident occurred | Correlation script in Task 5 fired the ALERT |
| **Contain** | Stop the attacker from doing more damage | iptables rule drops all packets from 203.0.113.9 |
| **Collect evidence** | Preserve proof for investigation/forensics | Copied auth.log with timestamp, hashed it to prove integrity |
| **Document** | Record what happened, when, and what was done | This incident report |

**Result:** The iptables rule blocks the attacker IP. The evidence copy is timestamped and its SHA-256 hash provides a means to verify the integrity of the evidence file by detecting subsequent modifications.

Evidence: <div align="left">
<img alt="Incident response — containment and evidence" src="evidence lab 5/lab5task6.png">
</div>

---

## Verification Command

```bash
aws --endpoint-url=http://localhost:4566 logs describe-log-groups
sha256sum -c evidence.sha256
```

```text
{
    "logGroups": [
        {
            "logGroupName": "C:/Program Files/Git/ccse/app",
            "creationTime": 1788347129920,
            "metricFilterCount": 0,
            "arn": "arn:aws:logs:us-east-1:000000000000:log-group:C:/Program Files/Git/ccse/app:*",
            "storedBytes": 397,
            "logGroupClass": "STANDARD",
            "logGroupArn": "arn:aws:logs:us-east-1:000000000000:log-group:C:/Program Files/Git/ccse/app"
        }
    ]
}

evidence_20260902.log: OK
```

**Note:** The log group name appears as `C:/Program Files/Git/ccse/app` instead of `/ccse/app` because Git Bash on Windows automatically converts paths starting with `/` to Windows paths. The log group was created and functions correctly — this is a cosmetic display difference.

**Why:** `describe-log-groups` confirms the CloudWatch log group exists and is accessible. `sha256sum -c` verifies the evidence file has not been tampered with since collection — `evidence_20260902.log: OK` confirms the hash matches.

Evidence: <div align="left">
<img alt="Verification commands output" src="evidence lab 5/lab5verify.png">
</div>

---

## Short-Answer Questions

**Q1. What is the difference between a log and an event? Give an example of each from this lab.**

| Aspect | Log | Event |
|---|---|---|
| **Definition** | A durable, timestamped record of something that happened | A trigger that fires in near real-time when a condition is met |
| **Storage** | Written to a file or centralised store (CloudWatch) | Generated by a monitoring system when a threshold is crossed |
| **Example from this lab** | `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9` (a line in auth.log) | `ALERT: probable brute-force -> compromise -> data exfiltration` (the correlation script in Task 5 firing when conditions matched) |

Logs are the raw data; events are the alerts derived from analysing that data. A log says "this happened"; an event says "this pattern means something is wrong."

---

**Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?**

Audit logs must be tamper-proof because an attacker's first move after a breach is to cover their tracks — deleting or modifying logs to hide what they did. If an attacker can alter the logs, they can erase evidence of the intrusion, making forensic investigation and compliance audits impossible.

A hash chain achieves tamper-evidence by linking each log entry to all previous entries through SHA-256 hashing. Each line's hash is computed from the previous hash plus the current line. If an attacker modifies any line (even a single character), the hash for that line changes, which cascades and changes every subsequent hash. The final hash acts as a "fingerprint" of the entire log — if it doesn't match, tampering has occurred.

In Task 4, changing `500MB` to `5MB` in the export line produced a completely different final hash, proving the chain detected the alteration.

---

**Q3. How did correlation detect an incident that no single log line revealed?**

No single log line in the attack sequence was alarming on its own:
- `LOGIN_FAIL` — could be a user who forgot their password
- `LOGIN_OK` — completely normal
- `EXPORT_DATA` — could be a legitimate business operation

But when correlated across time and source IP, the pattern tells a story:
1. Four rapid failures from `203.0.113.9` → brute-force attack
2. A success from the same IP → attacker guessed the password
3. A 500MB data export from the same IP → exfiltration after compromise

The correlation script in Task 5 detected this by counting: `fails ≥ 3 AND success ≥ 1 AND export ≥ 1` from the same IP. This is exactly what a SIEM does — it connects the dots across multiple log sources to produce a single, actionable alert that no individual log line would trigger.

---

**Q4. List the incident-response steps you performed and the goal of each.**

| Step | Goal | Action performed |
|---|---|---|
| **1. Detection** | Identify that an incident occurred | Correlation script in Task 5 detected brute-force → compromise → exfiltration pattern |
| **2. Containment** | Stop the attacker from causing further damage | Added iptables rule to DROP all traffic from 203.0.113.9 |
| **3. Evidence collection** | Preserve proof for investigation and potential legal action | Copied auth.log to timestamped evidence file, computed SHA-256 hash to prove integrity |
| **4. Documentation** | Record what happened, when, how it was detected, and what was done | This incident report — provides a timeline and lessons learned |

The goal is to move quickly to limit damage (contain), preserve proof (collect), and create a record for future reference (document). Detection is the trigger; the other three are the response.

---

**Q5. How do the same logs serve both security monitoring and compliance evidence (Weeks 6, 11)?**

The same centralised logs serve two distinct but related purposes:

**Security monitoring (detection):**
- Logs are queried in near real-time to detect suspicious activity
- Correlation scripts (Task 5) identify attack patterns
- Alerts trigger immediate response (Task 6 containment)
- Goal: stop active attacks and limit damage

**Compliance evidence (audit):**
- Logs are retained for months or years as proof of due diligence
- Tamper-proofing (Task 4 hash chain) ensures logs cannot be altered
- Centralisation (Task 2 CloudWatch) makes logs accessible to auditors
- Goal: prove to regulators that controls were in place and working

A single log line (`LOGIN_FAIL user=admin ip=203.0.113.9`) is useful for both: a security analyst sees it as a detection signal; an auditor sees it as proof that authentication failures are being recorded. The same infrastructure (centralised, tamper-proof logs) supports both use cases simultaneously.

---

## Incident Report

### Detection
The correlation script in Task 5 detected the incident by matching a pattern: 4+ failed login attempts from IP `203.0.113.9`, followed by a successful login, followed by a 500MB data export — all from the same source IP. This pattern indicates a brute-force attack that led to compromise and data exfiltration.

### Analysis
The attacker (`203.0.113.9`) performed a brute-force attack against the `admin` account, succeeding on the fifth attempt. After gaining access, they exported 500MB of data. The timeline spans from `09:01:10` to `09:01:40` — a 30-second window from first attempt to data theft.

### Containment
The attacker's IP was blocked immediately using an iptables DROP rule:
```
iptables -A INPUT -s 203.0.113.9 -j DROP
```
This prevents any further communication from the attacker to the system.

### Evidence & Integrity
- Original `auth.log` preserved as `evidence_20260902.log`
- SHA-256 hash computed and stored in `evidence.sha256`
- Hash-chained log (`auth.chain`) provides additional tamper-evidence
- All evidence stored with timestamps to establish chain of custody

### Lesson Learned
The attack succeeded because a single password was the only barrier. If MFA had been enabled (as in Lab 4), the brute-force would have failed even with the correct password. Additionally, the 500MB export should have triggered a data-loss-prevention (DLP) alert — monitoring for anomalous data volumes is a critical complement to login monitoring.

---

## Expansion — SOAR-Style Automated Response

A SOAR-style automated response script was implemented to automate the detection and containment process. The script counts failed login attempts from the suspicious IP address `203.0.113.9`. When the number of failed attempts reaches the threshold of three, it generates an alert and automatically executes an iptables DROP rule to block the attacker IP. This combines the correlation and containment steps from Tasks 5 and 6 into an automated response workflow, reducing the need for manual intervention.

```bash
cat auto_response.sh
```

```bash
#!/bin/bash

IP="203.0.113.9"
THRESHOLD=3

FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)

echo "Monitoring authentication logs..."
echo "IP=$IP"
echo "Failed attempts=$FAILS"
echo "Threshold=$THRESHOLD"

if [ "$FAILS" -ge "$THRESHOLD" ]; then
    echo "ALERT: Brute-force attack detected from $IP"
    echo "ACTION: Blocking attacker IP..."

    docker run --rm --cap-add=NET_ADMIN alpine sh -c \
    "apk add -q iptables; iptables -A INPUT -s $IP -j DROP; iptables -L INPUT -n | tail -2"

    echo "RESPONSE: IP $IP has been blocked."
else
    echo "No incident detected."
fi
```

```bash
chmod +x auto_response.sh
./auto_response.sh
```

**Result:**

```text
Monitoring authentication logs...
IP=203.0.113.9
Failed attempts=4
Threshold=3
ALERT: Brute-force attack detected from 203.0.113.9
ACTION: Blocking attacker IP...
target     prot opt source               destination
DROP       all  --  203.0.113.9          0.0.0.0/0
RESPONSE: IP 203.0.113.9 has been blocked.
```

**Why:** SOAR (Security Orchestration, Automation and Response) reduces mean-time-to-respond by automating the detection-to-containment pipeline. Instead of waiting for a human analyst to notice the correlation alert and manually run an iptables rule, the script detects the pattern and blocks the IP immediately — the same logic as Tasks 5 and 6, but without manual intervention.

Evidence: <div align="left">
<img alt="SOAR-style automated response script" src="evidence lab 5/lab5extra_soar.png">
</div>

---

## Cleanup & Teardown

```bash
# Remove generated files
rm -f auth.log auth.chain auth.tampered evidence_*.log evidence.sha256

# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```

**Why clean up?** These are disposable practice environments. Deleting them is itself good cloud hygiene — and keeps Docker from eating your disk.

## Security Best-Practices Checklist

- [x] Logs are centralised (CloudWatch), not left scattered on each host.
- [x] Security-relevant activity (failed logins) can be queried and grouped by IP.
- [x] Logs are tamper-evident (hash chain) and stored separately from the application.
- [x] An incident is detected by correlating multiple events (brute-force → compromise → exfiltration).
- [x] Incident response performed: detect, contain (iptables block), collect evidence (hash), document (this report).

---

## Conclusion

This lab demonstrated that monitoring, logging, and incident detection are the "eyes" of cloud security — without them, you are blind to attacks and unable to prove compliance.

### Session A — Logging & Centralisation
- Application logs were generated and shipped to CloudWatch Logs (LocalStack), proving that centralised logging enables cross-service querying.
- Failed login attempts were queried and grouped by IP, extracting security-relevant activity from raw logs.
- The distinction between a log (durable record) and an event (real-time trigger) was demonstrated.

### Session B — Tamper-Proofing, Detection & Response
- A hash-chained log proved that any modification — even a single character — breaks the chain and is detectable.
- Correlation detected an incident that no single log line revealed: brute-force → compromise → data exfiltration.
- Incident response was executed: containment (block IP), evidence collection (timestamped copy + hash), and documentation (this report).
- **Bonus (SOAR expansion):** An automated response script combined detection and containment into a single workflow, demonstrating how SOAR reduces mean-time-to-respond.

### Key Takeaway
You cannot secure — or prove compliance for — what you cannot see. Logs are foundational to detection, forensics, AND compliance evidence. The combination of centralised logging, tamper-proofing, correlation-based detection, and structured incident response creates a complete security monitoring capability — turning raw data into actionable intelligence and legally defensible evidence.
