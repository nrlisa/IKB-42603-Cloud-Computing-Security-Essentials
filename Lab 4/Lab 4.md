# Lab 4: Access Control & Network Security
Course: IKB42603 Cloud Computing Security Essentials

Topic: AuthN vs AuthZ, network segmentation and host hardening — Docker & Kubernetes

Environment: Docker, kind Kubernetes cluster ccse-lab4, oathtool for TOTP, Trivy scanner

Name: Nurlisa Sofiya binti Mahadzir

## Lab Summary // Objective

This lab demonstrates access control and network security across authentication, authorization, network segmentation, and container hardening:

- Authentication (AuthN) was implemented with HTTP Basic authentication, proving that unauthenticated requests are rejected with a 401.
- Multi-factor authentication (MFA) was added using TOTP, showing how a second factor defeats credential attacks.
- Authorization (AuthZ) was enforced using Kubernetes RBAC, proving that a developer role can list pods but cannot create or delete them.
- Network segmentation was implemented with three-tier Docker networks, proving that the frontend cannot reach the database directly.
- A default-deny firewall policy was configured with iptables, mirroring cloud security groups.
- Container hardening was applied (non-root, minimal, capabilities dropped, read-only filesystem) and the image was scanned for vulnerabilities with Trivy.

## Architecture Diagram

The following architecture models the access control and network security controls implemented across Sessions A and B.

```text
                   Access Control & Network Security (Lab 4)
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
     ┌────────┴────────┐   ┌───────┴───────┐   ┌────────┴────────┐
     │   Session A     │   │   Session B   │   │   Session B     │
     │  AuthN & AuthZ  │   │  Network Seg  │   │   Hardening     │
     │   Tasks 1–3     │   │  Tasks 4–5    │   │    Task 6       │
     └────────┬────────┘   └───────┬───────┘   └────────┬────────┘
              │                     │                     │
              │                     │                     │
   ┌──────────┴──────────┐ ┌───────┴────────┐  ┌────────┴────────┐
   │                     │ │                │  │                 │
   │  Auth service       │ │  frontend-net  │  │  hardened       │
   │  nginx :8080        │ │    │           │  │  container      │
   │                     │ │    ↓           │  │                 │
   │  ┌───────────────┐  │ │  web ──→ app  │  │  --user 1000    │
   │  │ HTTP Basic    │  │ │    │           │  │  --read-only    │
   │  │ Auth (401/200)│  │ │    ↓           │  │  --cap-drop ALL │
   │  └───────────────┘  │ │  BLOCKED       │  │  no-new-privs   │
   │                     │ │  web ✗→ db     │  │                 │
   │  ┌───────────────┐  │ │                │  │  ┌───────────┐  │
   │  │ TOTP MFA      │  │ │  backend-net   │  │  │ Trivy     │  │
   │  │ (oathtool)    │  │ │    │           │  │  │ Vulnerab- │  │
   │  └───────────────┘  │ │    ↓           │  │  │ ility     │  │
   │                     │ │  app ──→ db    │  │  │ Scan      │  │
   │  ┌───────────────┐  │ │  REACHABLE     │  │  └───────────┘  │
   │  │ RBAC Roles    │  │ │                │  │                 │
   │  │ can-i checks  │  │ │  Redis (db)    │  │  nginx-unpriv   │
   │  └───────────────┘  │ │                │  │                 │
   │                     │ │                │  │                 │
   └─────────────────────┘ └────────────────┘  └─────────────────┘
```

## Evidence Folder

All screenshots used for this report are stored in the `evidence lab 4` folder.

| **Evidence File** | **Purpose** |
|---|---|
| `lab4task1a.png` | HTTP Basic authentication — 401 (no credentials) result |
| `lab4task1b.png` | HTTP Basic authentication — 200 (valid credentials) result |
| `lab4task2a.png` | TOTP secret generation output |
| `lab4task2b.png` | MFA OK validation output |
| `lab4task3.png` | Kubernetes RBAC — developer role can-i list, create, delete pods results |
| `lab4task4a.png` | Three-tier network segmentation — web→db BLOCKED result |
| `lab4task4b.png` | Three-tier network segmentation — app→db REACHABLE result |
| `lab4task5.png` | iptables default-deny firewall ruleset |
| `lab4task6a.png` | Hardened container inspect output |
| `lab4task6b.png` | Trivy vulnerability scan summary |
| `lab4verify.png` | Verification commands output |

## Overview

This lab is split into two sessions:

- **Session A (Week 7):** Authentication vs authorization, MFA, and RBAC enforcement (Tasks 1–3). Controls WHO gets in.
- **Session B (Week 8):** Network segmentation, firewall rules and container hardening (Tasks 4–6), then the report. Controls WHAT they can reach and reduces WHAT an intruder could exploit.

Session A controls WHO gets in. Session B controls WHAT they can reach and reduces WHAT an intruder could exploit.

**Security tip:** Identity is the perimeter. Notice that almost every control in this lab ultimately asks the same two questions: are you who you claim, and are you allowed to do this?

---

## Session A (Week 7) — Authentication & Authorization

### Task 1 — Authentication: a Password-Protected Service

Run a web service behind HTTP Basic authentication. Only requests with valid credentials get in.

```bash
# Create a password file (user: student)
docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt

# Serve a page that requires authentication
cat > default.conf <<'EOF'
server {
  listen 80;
  location / {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    return 200 'Authenticated OK\n';
  }
}
EOF

docker run --rm -d --name authsvc -p 8080:80 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
  -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd nginx

curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080   # 401
curl -s -u student:'P@ssw0rd!' http://localhost:8080                        # 200
Authenticated OK
```

**Why:** HTTP Basic authentication requires a valid username:password pair. Without credentials, the server returns `401 Unauthorized`. With valid credentials, it returns `200 OK` and the protected content. This is the simplest form of authentication — proving identity before allowing access.

**Results:**
- Without credentials → `401` (blocked)
- With valid credentials → `200` (allowed), body: `Authenticated OK`

Evidence: <div align="left">
<img alt="HTTP Basic authentication 401 result" src="evidence lab 4/lab4task1a.png">
<img alt="HTTP Basic authentication 200 result" src="evidence lab 4/lab4task1b.png">
</div>

---

### Task 2 — Add a Second Factor (MFA / TOTP)

Passwords alone are weak. Generate a time-based one-time password and validate it, the same mechanism an authenticator app uses.

```bash
# Create a shared secret (base32) and generate the current 6-digit code
SECRET=$(head -c20 /dev/urandom | base32)
echo "Enrol this secret in an authenticator app: $SECRET"
oathtool --totp -b "$SECRET"

# Validate a code the user types (compare to the expected value)
read -p 'Enter the 6-digit code: ' CODE
[ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
```

**Why:** MFA combines factors from different classes (something you know + something you have). It defeats the majority of credential attacks — the cheapest big security win. Even if an attacker steals the password, they still need the physical device generating the TOTP code.

**Result:** `MFA OK` when the correct 6-digit code is entered.

**Note:** MFA is the single most effective control against account takeover attacks, blocking credential stuffing, phishing, and password reuse. The TOTP code rotates every 30 seconds, so even an intercepted code is only valid for a brief window.

Evidence: <div align="left">
<img alt="TOTP secret generation" src="evidence lab 4/lab4task2a.png">
<img alt="MFA OK validation" src="evidence lab 4/lab4task2b.png">
</div>

---

### Task 3 — Authorization: RBAC Roles

Authentication proves identity; authorization decides permissions. Create a cluster and compare a developer role with an admin role.

```bash
kind create cluster --name ccse-lab4

kubectl create namespace app
kubectl create serviceaccount dev -n app

# Developer may only read pods
kubectl create role dev-role -n app --verb=get,list --resource=pods
kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev

SA=system:serviceaccount:app:dev
kubectl auth can-i list pods    -n app --as=$SA   # yes
kubectl auth can-i create deploy -n app --as=$SA   # no
kubectl auth can-i delete pods  -n app --as=$SA   # no
```

**Why:** The `dev-role` grants only `get` and `list` on pods — read-only access. Testing with `can-i` proves the RBAC boundary without actually performing the actions. A Role does nothing until it is bound to an identity via a RoleBinding.

**Results:**
- `list pods -n app` → `yes` — allowed because `dev-role` grants `list` on pods in `app`.
- `create deploy -n app` → `no` — blocked because the role only grants `get` and `list`; `create` was never granted.
- `delete pods -n app` → `no` — blocked because the role does not include `delete`.

RBAC allows only explicitly permitted actions, even within the same cluster.

**Note:** End of Session A. Keep the 401/200 results, the MFA OK output, and the three can-i results. Stop the auth service (`docker stop authsvc`).

Evidence: <div align="left">
<img alt="RBAC authorization test results" src="evidence lab 4/lab4task3.png">
</div>

*End of Session A. Session B moves to network security and container hardening.*

---

## Session B (Week 8) — Network Security & Hardening

### Task 4 — Network Segmentation (Three-Tier)

Separate a frontend, backend and database into isolated Docker networks so the frontend cannot reach the database directly — defence in depth for the network.

```bash
# Create two segmented networks
docker network create frontend-net
docker network create backend-net

# DB only on backend-net; app on both; web only on frontend-net
docker run -d --name db  --network backend-net  redis:alpine
docker run -d --name app --network backend-net  nginx
docker network connect frontend-net app
docker run -d --name web --network frontend-net nginx

# web -> db should FAIL (not on the same network)
docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
# app -> db should WORK (shared backend-net)
docker exec app sh -c 'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
```

**Why:** Network segmentation implements the principle of least privilege for network access. The database is unreachable from the internet-facing tier. An attacker who compromises the web tier still cannot talk directly to the data — segmentation contains lateral movement.

**Results:**
- `web → db` → `BLOCKED` (different networks)
- `app → db` → `REACHABLE` (shared backend-net)

**Security tip:** The database is unreachable from the internet-facing tier. An attacker who compromises the web tier still cannot talk directly to the data — segmentation contains lateral movement.

Evidence: <div align="left">
<img alt="web→db BLOCKED" src="evidence lab 4/lab4task4a.png">
<img alt="app→db REACHABLE" src="evidence lab 4/lab4task4b.png">
</div>

---

### Task 5 — Firewall Rules (Default-Deny)

Apply host-level firewall rules that permit only the ports you need. This mirrors cloud security groups.

```bash
# Inside a throwaway container with iptables, model default-deny + allow 443
docker run --rm --cap-add=NET_ADMIN alpine sh -c '\
  apk add -q iptables; \
  iptables -P INPUT DROP; \
  iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
  iptables -A INPUT -i lo -j ACCEPT; \
  iptables -L INPUT -n'
```

**Why:** Default policy DROP with a single explicit ACCEPT is the security-group model: nothing is allowed unless you permit it (least privilege for the network). This is exactly how AWS Security Groups, Azure NSGs, and GCP firewall rules work.

**Result:** The iptables ruleset shows `DROP` default policy with explicit `ACCEPT` for port 443 and loopback traffic.

Evidence: <div align="left">
<img alt="iptables default-deny ruleset" src="evidence lab 4/lab4task5.png">
</div>

---

### Task 6 — Container / Host Hardening

Reduce the attack surface. Build a minimal, non-root, capability-dropped, read-only container and scan it.

```bash
# A hardened run of a service
docker run -d --name hardened \
  --user 1000:1000 \            # non-root
  --read-only \                 # read-only root filesystem
  --cap-drop=ALL \             # drop all Linux capabilities
  --security-opt no-new-privileges \
  --tmpfs /tmp \
  nginxinc/nginx-unprivileged

docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'

# Scan an image for known vulnerabilities
docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
```

**Why:** Each hardening measure reduces the attack surface independently:

| Measure | Attack Surface Removed |
|---|---|
| `--user 1000:1000` (non-root) | Prevents privilege escalation to root — an attacker who exploits the container is still an unprivileged user |
| `--read-only` (read-only filesystem) | Prevents malware from writing tools, downloading payloads, or modifying configurations |
| `--cap-drop=ALL` (no capabilities) | Removes Linux capabilities (e.g., `NET_ADMIN`, `SYS_ADMIN`) that could be used for network attacks or container escape |
| `--security-opt no-new-privileges` | Prevents gaining new privileges via setuid/setgid binaries inside the container |

**Result:** Container runs as non-root (`User=1000:1000`) with read-only filesystem (`ReadOnly=true`) and no capabilities. Trivy scan identifies known vulnerabilities in the nginx:alpine image.

**In your report, list three hardening measures you applied and the attack each one blunts:**

1. **Non-root (`--user 1000:1000`)** — Blunts privilege escalation. Even if an attacker exploits a vulnerability in nginx, they only gain an unprivileged user's permissions, not root. They cannot modify system files, install kernel modules, or escape the container via root-level exploits.

2. **Read-only filesystem (`--read-only`)** — Blunts persistence and tool dropping. Malware cannot write backdoors, download additional tools, modify nginx configuration, or tamper with application files. The attacker is limited to what exists in the image — no writes means no foothold.

3. **Capability drop (`--cap-drop=ALL`)** — Blunts container escape and network abuse. Without `NET_ADMIN`, the attacker cannot sniff traffic or modify firewall rules. Without `SYS_ADMIN`, they cannot mount filesystems or exploit kernel attack surfaces for container escape. The container has the minimum kernel privileges needed to run nginx.

Evidence: <div align="left">
<img alt="Hardened container inspect output" src="evidence lab 4/lab4task6a.png">
<img alt="Trivy vulnerability scan summary" src="evidence lab 4/lab4task6b.png">
</div>

---

## Verification Command

```bash
kubectl get rolebinding dev-rb -n app -o yaml
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
```

**Why:** Confirms the RBAC binding is correctly in place and that the container has dropped all capabilities as configured.

**Result:** RoleBinding YAML shows `dev-role` bound to `dev` service account in `app` namespace. CapDrop shows `["ALL"]`.

Evidence: <div align="left">
<img alt="Verification commands output" src="evidence lab 4/lab4verify.png">
</div>

---

## Short-Answer Questions

**Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.**

| Aspect | Authentication (Task 1) | Authorization (Task 3) |
|---|---|---|
| **Question answered** | Who are you? | What may you do? |
| **Mechanism** | HTTP Basic auth (username + password) | RBAC (Role + RoleBinding) |
| **Test** | 401 without credentials, 200 with valid credentials | `can-i` checks for list, create, delete |
| **Failure mode** | Invalid credentials → 401 | Valid identity but denied action → `no` |

Authentication proves identity; authorization enforces permissions. A user can authenticate successfully but still be denied authorization for specific actions. Task 1 proves you are who you claim (username + password); Task 3 proves what you are allowed to do (read-only pods in the `app` namespace).

**Q2. Why is MFA so effective, and which attacks does it defeat?**

MFA is effective because it requires two different factor classes: something you know (password) and something you have (TOTP device). This defeats:
- **Credential stuffing** — stolen passwords are useless without the second factor
- **Phishing** — even if the password is phished, the attacker needs the TOTP code
- **Brute force** — guessing the password doesn't help without the second factor
- **Password reuse** — leaked passwords from other sites can't be used without the second factor

MFA is the single most cost-effective security control, blocking the vast majority of account takeover attacks. The TOTP code rotates every 30 seconds, so even an intercepted code is only valid for a brief window.

**Q3. How does network segmentation limit the damage of a compromised web server?**

In Task 4, the `web` tier was on `frontend-net` only, while `db` was on `backend-net` only. The `app` tier bridged both networks. If an attacker compromises the web server, they cannot reach the database because the networks are isolated — the `curl -s -m 3 db:6379` command returned `BLOCKED`.

The attacker is contained to the frontend network and cannot move laterally to the data tier. This is defence in depth: even with a successful web-tier compromise, the blast radius is limited to what that specific tier can reach. The database remains unreachable from the internet-facing tier.

**Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?**

A default-deny policy (`iptables -P INPUT DROP`) blocks all traffic by default and only allows explicitly permitted ports (e.g., port 443). This mirrors cloud security groups (AWS Security Groups, Azure NSGs, GCP firewall rules) which operate on the same principle: deny all inbound/outbound by default, then add rules for specific ports and sources.

Both implement least privilege for network access — nothing is allowed unless explicitly permitted. In Task 5, the ruleset showed `DROP` as the default policy with only port 443 and loopback allowed — the same model used in production cloud environments.

**Q5. List the hardening measures you applied and the attack surface each one removes.**

| Measure | Attack Surface Removed |
|---|---|
| `--user 1000:1000` (non-root) | Prevents privilege escalation to root — an attacker who exploits the container is still an unprivileged user and cannot modify system files or processes |
| `--read-only` (read-only filesystem) | Prevents malware from writing to the filesystem, downloading tools, persisting backdoors, or modifying configurations |
| `--cap-drop=ALL` (no capabilities) | Removes Linux capabilities (`NET_ADMIN`, `SYS_ADMIN`, etc.) that could be used for network sniffing, packet injection, or container escape |
| `--security-opt no-new-privileges` | Prevents gaining new privileges via setuid/setgid binaries inside the container, closing a common privilege escalation vector |

**Key Takeaway:** Each measure independently reduces what an attacker can do, making successful exploitation far less useful. Together they create defence in depth for the container runtime.

---

## Cleanup & Teardown

```bash
# Remove containers
docker rm -f authsvc db app web hardened 2>/dev/null

# Remove networks
docker network rm frontend-net backend-net 2>/dev/null

# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab4
```

**Why clean up?** These are disposable practice environments. Deleting them is itself good cloud hygiene — and keeps Docker from eating your disk.

## Security Best-Practices Checklist

- [x] Service requires authentication (unauthenticated requests rejected with 401).
- [x] MFA / second factor implemented and validated (TOTP code verified, MFA OK).
- [x] Authorization enforced by RBAC (least privilege; unauthorized actions denied).
- [x] Network segmented so the data tier is unreachable from the front tier.
- [x] Default-deny firewall with explicit allow rules.
- [x] Container hardened: non-root, minimal, capabilities dropped, read-only; image scanned with Trivy.

---

## Conclusion

This lab demonstrated that access control and network security require multiple layers: authentication proves identity, authorization enforces permissions, network segmentation limits lateral movement, and container hardening reduces the attack surface.

### Session A — Authentication & Authorization
- HTTP Basic authentication rejected unauthenticated requests (401) and allowed valid credentials (200).
- TOTP-based MFA provided a second factor that defeats credential attacks.
- Kubernetes RBAC enforced least privilege: the developer role could list pods but not create or delete them.

### Session B — Network Security & Hardening
- Three-tier network segmentation isolated the database from the frontend, containing lateral movement.
- Default-deny iptables rules mirrored cloud security groups — nothing allowed unless explicitly permitted.
- Container hardening (non-root, read-only, no capabilities) reduced the attack surface, and Trivy identified known vulnerabilities.

### Key Takeaway
Identity is the perimeter. Every control in this lab answers two questions: **Are you who you claim?** and **Are you allowed to do this?** The combination of authentication, authorization, network segmentation, and hardening creates defence in depth — no single control is sufficient, but together they dramatically reduce risk.
