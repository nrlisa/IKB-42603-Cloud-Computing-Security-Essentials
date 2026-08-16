# Lab 2 — Commands Explained (Easy Words)

A plain-words guide to every command in **Lab 2** (`Lab 2.md`, plus the Extra Task 7 file). It's split into the same sections as the report.

## Big Picture

Lab 2 is about **multi-tenant isolation** — proving that two customers ("tenants") sharing one cloud can't see or break each other:

- **Session A (Week 3)** — Compute isolation: two tenants in separate namespaces on one cluster, watching the **default-open risk**, then containing them with a ResourceQuota.
- **Session B (Week 4)** — Network & storage isolation: a default-deny NetworkPolicy, per-tenant secrets locked down with RBAC, and data remanence (deleted ≠ gone).
- **Extra Task 7** — Egress default-deny: block all outgoing traffic, then allow only exactly what's needed (DNS + one service) — micro-segmentation.

The whole lab follows one rhythm: **show the problem → apply the control → re-run the same test to prove the change**.

---

## Environment Setup — build the cluster with "network police" (Calico)

| Command | What it does, in easy words |
|---|---|
| `cat <<EOF \| kind create cluster --name ccse-lab2 --config=-` | Creates a disposable Kubernetes cluster in Docker called `ccse-lab2`, but with a custom config file. The config turns **off** the default networking plugin and sets a private pod network. (The default plugin can't enforce NetworkPolicy — the "walls" between tenants.) |
| `kubectl apply -f https://...calico.yaml` | Installs **Calico**, the network police force. Calico is the plugin that can actually enforce NetworkPolicy rules. Without it, Task 4's wall wouldn't hold. |
| `kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s` | "Wait until every Calico officer is on duty." `rollout status` blocks until the `calico-node` agents on all nodes report ready, or gives up after 180 seconds. |

**Why:** kind's default networking ignores NetworkPolicy. The lab needs real policy enforcement, so Calico is installed first.

---

## Task 1 — Two Tenants on One Cluster

| Command | What it does |
|---|---|
| `kubectl create namespace tenant-a` / `tenant-b` | Makes two separate **rooms** for two different tenants on the same cluster. Each room gets its own toys (pods, services, secrets) that are logically kept apart. |
| `kubectl -n tenant-a create deployment web --image=nginx` | Deploys a simple web server (nginx) inside `tenant-a`'s room. `-n tenant-a` says "which room". |
| `kubectl -n tenant-b create deployment web --image=nginx` | Same web server for `tenant-b`. |
| `kubectl -n tenant-a expose deployment web --port=80` | Gives the web server a **phone number** (a Service). Other pods can now dial it by name/IP instead of hunting for the pod. |
| `kubectl -n tenant-b expose deployment web --port=80` | Same phone number for `tenant-b`'s web server. |
| `kubectl get pods,svc -n tenant-a` | Lists the pods and services in `tenant-a` — checks the pod is `Running` and the Service got a ClusterIP before testing anything. |

**Why:** two separate tenants on one cluster = the setup that makes the cross-tenant test in Task 2 possible.

---

## Task 2 — Observe the Default-Open Risk

| Command | What it does |
|---|---|
| `kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo` | Finds `tenant-b`'s web server phone number (its ClusterIP). `-o jsonpath` says "extract just the IP field" so you can paste it into the next command; `echo` adds a newline. |
| `kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'` | From inside `tenant-a`'s room, dials `tenant-b`'s web server. `--rm` = delete the probe pod when done; `-it` = interactive terminal; `--image=curlimages/curl` = a tiny container with `curl` installed; `-m 5` = give up after 5 seconds; `-o /dev/null` = throw away the page content; `-w 'HTTP %{http_code}'` = print only the status code. |

**Why:** Kubernetes namespaces are just **labels**, not walls. Nothing stops a pod in one room from dialing a pod in another — the probe proves it with `HTTP 200`.

**The point:** namespaces give logical separation, not network isolation. That's the vulnerability the next tasks fix.

---

## Task 3 — Contain the Noisy Neighbour (Resource Quota)

| Command | What it does |
|---|---|
| `cat <<EOF \| kubectl apply -f -` + ResourceQuota YAML | Feeds a quota form to the cluster: `tenant-a` is allowed at most **5 pods, 1 CPU, 512Mi memory**. Like setting a spending limit on a shared credit card. |
| `kubectl describe resourcequota tenant-a-quota -n tenant-a` | Shows the quota's limits and how much is currently used — proof the limit is in place and enforced. |

**Why:** isolation isn't only about walls — one greedy tenant could hog all CPU/memory and slow everyone else down (the "noisy neighbour" problem). The quota caps how much one tenant can take.

---

## Session B — Network & Storage Isolation

## Task 4 — Default-Deny Network Isolation

| Command | What it does |
|---|---|
| `cat <<EOF \| kubectl apply -f -` + NetworkPolicy YAML (`default-deny-ingress`) | Builds a wall around `tenant-b`. `podSelector: {}` = applies to **every** pod in the room; `policyTypes: [Ingress]` with no rules = **no one is allowed in**. Traffic is blocked unless a rule explicitly says otherwise. |
| Re-run the SAME probe from Task 2 | The exact same curl from `tenant-a` to `tenant-b` — but now it **times out** instead of returning `HTTP 200`. |

**Why:** this is the before/after proof. Same command, same network, different result — only the policy changed. That's how you know the wall works: `HTTP 200` → timeout.

---

## Task 5 — Storage & Secret Isolation (RBAC)

| Command | What it does |
|---|---|
| `kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A` | Stores a secret (like a password) inside `tenant-a`'s room. `--from-literal` = "here's the value directly". |
| `kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B` | Same for `tenant-b` — a different secret. |
| `kubectl -n tenant-a create serviceaccount app-a` | Creates a **robot identity** (`app-a`) that represents `tenant-a`'s application. |
| `kubectl -n tenant-a create role reader --verb=get --resource=secrets` | Writes the job description: may only **read** (`get`) secrets — nothing else. |
| `kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a` | Hires `app-a` for that job inside `tenant-a`. A role does nothing until it's bound to an identity. |
| `SA=system:serviceaccount:tenant-a:app-a` | Stores the robot's **full address** in a variable, so the can-i checks below can pretend to be it. |
| `kubectl auth can-i get secrets -n tenant-a --as=$SA` | "Can this robot read secrets in tenant-a?" → **yes** (explicitly allowed). |
| `kubectl auth can-i get secrets -n tenant-b --as=$SA` | "Can it read secrets in tenant-b?" → **no** — no permission exists there, even though it's the same cluster. |

**Why:** secrets are like keys in a drawer. The robot can open its own drawer (`tenant-a`) but the other tenant's drawer is locked — proving storage isolation with the same `auth can-i` trick from Lab 1.

---

## Task 6 — Data Remanence & Secure Deletion

| Command | What it does |
|---|---|
| `docker run --rm -v ccse-vol:/data alpine sh -c 'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'` | Starts a tiny Linux container, mounts a disk volume (`ccse-vol`) at `/data`, then runs a mini-script: **write** a sensitive file → **flush** it to disk (`sync`) → **delete** it normally (`rm`) → **scan** the disk (`grep`) to see if the bytes survived. `--rm` = remove the container when done; `-v ccse-vol:/data` = attach the volume; `alpine` = tiny Linux image; `sh -c '…'` = run the quoted script. |
| `docker run --rm -v ccse-vol:/data alpine sh -c 'echo SENSITIVE > /data/phi2.txt; sync; dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo wiped'` | Same idea, but this time **overwrite** the file with zeros first (`dd if=/dev/zero`) before deleting it — a secure wipe. |

**Why:** `rm` only removes the "address label" for the file; the underlying bytes often stay on disk. The first command proves the deleted secret is still readable (**data remanence**); the second shows that overwriting before delete reduces the risk. In real clouds, you can't touch the physical disk, so **cryptographic erasure** (destroy the key, not the bytes) is the practical answer.

---

## Extra Task 7 — Egress Default-Deny & Micro-Segmentation

| Command | What it does |
|---|---|
| `kubectl get pods -n tenant-b --show-labels` | Checks the web pod's **name tags** (labels). The `app=web` label is the address used to allow traffic to exactly this workload. |
| `kubectl get svc web -n tenant-b` | Verifies the target service exists and exposes port 80. |
| NetworkPolicy `default-deny-egress` (in `tenant-a`) | Blocks **all outgoing** traffic from `tenant-a`'s pods — deny by default. |
| NetworkPolicy `allow-dns-egress` (in `tenant-a`) | First exception: allows DNS (UDP/TCP port 53) to the cluster's DNS pods in `kube-system`, so apps can still resolve names. |
| NetworkPolicy `allow-tenant-b-web` (in `tenant-a`) | Second exception: allows outbound traffic **only** to pods labelled `app=web` in `tenant-b`, and only on TCP port 80. |
| NetworkPolicy `allow-tenant-a-to-web` (in `tenant-b`) | The matching "welcome mat": since `tenant-b` has default-deny *ingress*, this rule lets the traffic from `tenant-a` actually arrive at the web pod. |
| `kubectl -n tenant-a run probe-allowed --rm -it --image=curlimages/curl --restart=Never -- curl -s -m 5 http://10.96.207.129 -o /dev/null -w 'HTTP %{http_code}\n'` | The test: dial the web service through all the walls. Returns `HTTP 200` — the specifically allowed path still works. |
| `kubectl -n tenant-a run dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup kubernetes.default.svc.cluster.local` | The DNS test: resolves a service name through the cluster DNS — proves the DNS exception works. |
| `kubectl get networkpolicy -A` | The final photo of every policy across all namespaces — the complete micro-segmentation map. |

**Why:** blocking all egress would break the app — it needs DNS and one service. So the goal is **deny by default, then explicitly allow only what's required** — that's micro-segmentation.

---

## Verification Command

| Command | What it does |
|---|---|
| `kubectl get networkpolicy -A` | Lists every NetworkPolicy in every namespace — proof the walls (`default-deny-ingress` in `tenant-b`, the three egress policies in `tenant-a`) are still standing at the end of the lab. |
| `kubectl describe resourcequota tenant-a-quota -n tenant-a` | Shows the quota's hard limits and current usage — proof the compute limit is still enforced. |

---

## Cleanup

| Command | What it does |
|---|---|
| `kind delete cluster --name ccse-lab2` | Destroys the whole practice cluster (all rooms, walls, and robots). |
| `docker volume rm ccse-vol` | Removes the disk volume that held the leftover demo data — good hygiene, and the right instinct after a data-remanence lesson. |

---

## One-line takeaway

Isolation on shared infrastructure is **not automatic** — it must be deliberately built and *tested* across three dimensions: **compute** (namespaces + ResourceQuota), **network** (default-deny NetworkPolicy), and **storage** (RBAC on secrets + secure deletion). Every control was proven with a before/after comparison: `HTTP 200` → timeout, `yes` → `no`, bytes found → wiped.
