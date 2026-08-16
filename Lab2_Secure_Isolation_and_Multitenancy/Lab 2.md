# Lab 2: Secure Isolation & Multi-Tenancy
Course: IKB42603 Cloud Computing Security Essentials

Topic: Compute, network and storage isolation, default-deny NetworkPolicy, RBAC secret isolation, data remanence

Environment: kind Kubernetes cluster ccse-lab2 (Calico CNI) and Docker

Name: Nurlisa Sofiya binti Mahadzir

## Lab Summary // Objective

This lab demonstrates multi-tenant isolation on shared cloud infrastructure using Kubernetes and Docker:

Compute isolation was modelled by placing two simulated tenants — `tenant-a` and `tenant-b` — into separate namespaces on the same cluster, and containing resource usage with a ResourceQuota.
Network isolation was enforced with a default-deny Kubernetes NetworkPolicy, proving that cross-tenant traffic which succeeded by default was blocked once the policy was applied.
Storage isolation was enforced with Kubernetes RBAC, proving that a service account scoped to one tenant cannot read another tenant's Secret.
Data remanence was explored using a Docker volume, showing that a normally deleted file can still leave recoverable bytes on disk, and that a secure "wipe before delete" reduces this risk.

## Architecture Diagram

The following architecture models two mutually untrusted tenants sharing the
same Kubernetes cluster. The lab evaluates isolation across compute, network,
and storage dimensions.

```text
                     Kubernetes Cluster
                     ccse-lab2 / Calico
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
    Namespace: tenant-a                Namespace: tenant-b
          │                                   │
      nginx web                         nginx web
          │                                   │
    ResourceQuota                  default-deny NetworkPolicy
          │                                   │
    ServiceAccount ──RBAC──→ Secrets        Secrets
          │                                   │
          └────────── X Network ──────────────┘
```
## Evidence Folder

All screenshots used for this report are stored in the `evidence lab2` folder.

| **Evidence File** | **Purpose** |
|---|---|
| `setup1.png` | `kind create cluster --config=-` output showing cluster `ccse-lab2` created with default CNI disabled |
| `setup2.png` | Calico manifest applied and `calico-node` DaemonSet rollout status confirming policy enforcement is active |
| `task1.png` | `tenant-a` and `tenant-b` namespace creation, web deployment and service creation in both namespaces, and `kubectl get pods,svc -n tenant-a` output |
| `task2.png` | `tenant-b` service ClusterIP lookup and cross-namespace probe from `tenant-a` to `tenant-b` returning `HTTP 200` (default-open risk) |
| `task3.png` | `ResourceQuota` `tenant-a-quota` applied and `kubectl describe resourcequota tenant-a-quota -n tenant-a` output |
| `task4.png` | `default-deny-ingress` NetworkPolicy applied to `tenant-b` and same probe re-run after policy application, now timing out (network isolation proven) |
| `task5.png` | Secrets created in `tenant-a` and `tenant-b`, service account, Role and RoleBinding creation for `app-a`, and `kubectl auth can-i` results proving secret isolation |
| `task6.png` | Data remanence scan showing `SENSITIVE-PATIENT-RECORD` still recoverable after normal `rm`, and secure wipe (`dd` overwrite) before delete confirming mitigation |
| `verify.png` | `kubectl get networkpolicy -A` and `resourcequota` verification output |
## Overview

This lab is split into two sessions:

- **Session A (Week 3):** Compute isolation — two tenants deployed into separate namespaces on one cluster, observing the default-open risk between them, and containing resource usage with a quota (Tasks 1–3).
- **Session B (Week 4):** Network and storage isolation — applying a default-deny NetworkPolicy, proving per-tenant secret isolation via RBAC, and demonstrating data remanence and secure deletion (Tasks 4–6).

Session A shows the problem: **isolation is not automatic** on shared infrastructure. Session B applies **NetworkPolicy and RBAC** to enforce separation.

The cluster uses **Calico** as its CNI because the default kind CNI does not enforce `NetworkPolicy` objects. Without Calico, the network isolation test would not work as intended.

---
## Session A (Week 3) — Compute Isolation & the Default-Open Risk

## Environment Setup

```bash
# Create a cluster with the default CNI disabled, then install Calico
cat <<EOF | kind create cluster --name ccse-lab2 --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
EOF

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```
**Why:** The default kind CNI does not enforce `NetworkPolicy`. Calico provides the policy enforcement needed for Task 4.

**Result:** The cluster `ccse-lab2` was created successfully, Calico was installed, and the `calico-node` DaemonSet rolled out successfully.
Evidence: <div align="left">
<img alt="kind cluster creation with Calico CNI" src="evidence lab2/setup1.png">
<img alt="Calico rollout status" src="evidence lab2/setup2.png">
</div>

---

## Task 1 — Two Tenants on One Cluster

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```
**Why:** Namespaces model two separate tenants sharing the same cluster.
**Result:** The namespaces `tenant-a` and `tenant-b` were created successfully.

```bash
# Deploy a simple web server for each tenant
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
```
**Why:** Creates a workload and Service for each tenant so cross-tenant access can be tested in Task 2.
**Result:** A `web` Deployment and Service were created in both `tenant-a` and `tenant-b`.

```bash
kubectl get pods,svc -n tenant-a
```
**Why:** Confirms the pod is `Running` and the Service has a ClusterIP allocated before attempting to reach it from the other tenant.

Evidence: <div align="left">
<img alt="Namespace creation" src="evidence lab2/task1.png">
</div>

---

## Task 2 — Observe the Default-Open Risk

```bash
# Get tenant-b's service IP
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo

# From tenant-a, curl tenant-b (replace <B_IP>)
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```
**Why:** Kubernetes does not provide network isolation between namespaces by default. A pod can reach other pods or Services unless a `NetworkPolicy` blocks the traffic.

**Result:** The probe returned `HTTP 200`, proving `tenant-a` could reach `tenant-b` across the namespace boundary.
Namespaces provide **logical separation**, not network isolation. `NetworkPolicy` is required to restrict cross-tenant traffic.

Evidence: <div align="left">
<img alt="tenant-b ClusterIP lookup" src="evidence lab2/task2.png">
</div>

---

## Task 3 — Contain the Noisy Neighbour (Resource Quotas)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF

kubectl describe resourcequota tenant-a-quota -n tenant-a
```
**Why:** Resource isolation also includes shared capacity. Without quotas, one tenant could consume excessive CPU, memory, or pods and affect other tenants.

**Result:** `tenant-a-quota` was created with these limits:

| Resource | Limit |
|---|---:|
| Pods | `5` |
| CPU requests | `1` |
| Memory requests | `512Mi` |

`kubectl describe` confirmed the quota is enforced for new resources in `tenant-a`.
`ResourceQuota` limits tenant resource consumption and helps prevent **noisy-neighbour** problems.

Evidence: <div align="left">
<img alt="ResourceQuota applied" src="evidence lab2/task3.png">
</div>

*End of Session A. The `HTTP 200` result from Task 2 was retained to compare against Session B, where the same probe is re-run after a NetworkPolicy is applied.*

---

## Session B (Week 4) — Network & Storage Isolation

## Task 4 — Default-Deny Network Isolation

```bash
# Deny ALL ingress into tenant-b
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

# Re-run the SAME probe from Task 2 — it should now TIME OUT / fail
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```
**Why:** `podSelector: {}` selects all pods in `tenant-b`. With no `ingress` rules, all incoming traffic is denied by default. Re-running the same probe provides a clear before/after comparison.

**Result:**
- Task 2 → `HTTP 200` — traffic allowed.
- Task 4 → **Timeout/fail** — traffic blocked.

This confirms that `NetworkPolicy` provides network isolation that namespaces alone do not.

Evidence:

**Before:**
<div align="left">
<img alt="Default-deny NetworkPolicy" src="evidence lab2/task2.png">
</div>

**After:**
<div align="left">
<img alt="Default-deny NetworkPolicy applied" src="evidence lab2/task4.1.png">
<img alt="Probe timing out after policy applied" src="evidence lab2/task4.2.png">
</div>

---

## Task 5 — Storage & Secret Isolation

```bash
# Create a secret in each tenant
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B

# A service account scoped to tenant-a only
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a

SA=system:serviceaccount:tenant-a:app-a
kubectl auth can-i get secrets -n tenant-a --as=$SA   # expect: yes
kubectl auth can-i get secrets -n tenant-b --as=$SA   # expect: no
```
**Why:** `app-a` only has `get` access to Secrets in `tenant-a` through a `Role` + `RoleBinding`. Testing both namespaces verifies that RBAC enforces storage isolation.

**Result:**
- `get secrets -n tenant-a` → `yes` — explicitly allowed.
- `get secrets -n tenant-b` → `no` — denied because no permission exists in `tenant-b`.

This confirms that `tenant-a`'s workload cannot read `tenant-b`'s secrets, even in the same cluster.

Evidence: <div align="left">
<img alt="Secrets created per tenant" src="evidence lab2/task5.1.png">
<img alt="Service account, role and rolebinding creation" src="evidence lab2/task5.2.png">
</div>

---

## Task 6 — Data Remanence & Secure Deletion

```bash
# Create a file, delete it normally, then show the bytes may persist
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```
**Why:** `rm` removes the filesystem reference but may not overwrite the underlying bytes. The scan checks if the deleted data is still recoverable.

**Result:** `grep` found `SENSITIVE-PATIENT-RECORD` after deletion, demonstrating **data remanence**.

```bash
# Secure wipe: overwrite before delete (shred)
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE > /data/phi2.txt; sync; \
dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
echo wiped'
```
**Why:** `dd` overwrites the file with zeros before deletion, reducing the chance of data recovery.

**Result:** The sensitive content was overwritten before deletion. In real cloud storage, **cryptographic erasure** is more practical because storage is usually virtualized, shared, and replicated.

Evidence: <div align="left">
<img alt="Data remanence scan showing recoverable content" src="evidence lab2/task6.png">
</div>

---

## Verification Command

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```
**Why:** Dumps the cluster-wide list of `NetworkPolicy` objects and the current status of the `ResourceQuota`, providing evidence that both the network-isolation and compute-isolation controls are still correctly in place at the end of the lab.
**Result:** `default-deny-ingress` was confirmed present in `tenant-b`, and `tenant-a-quota` showed the configured hard limits (`requests.cpu: 1`, `requests.memory: 512Mi`, `pods: 5`) with current usage beneath them.

Evidence: <div align="left">
<img alt="NetworkPolicy and ResourceQuota verification" src="evidence lab2/verify1.png">
</div>
<div align="left">
<img alt="NetworkPolicy and ResourceQuota verification" src="evidence lab2/verify2.png">
</div>

---

### Extra task: Task 7 — Egress Default-Deny & Micro-Segmentation

> Full Task 7 evidence is documented separately:
> [View Task 7 — Egress Default-Deny & Micro-Segmentation](Extra%20task-%20Egress%20Default-Deny%20%26%20Micro-Segmentation.md)

## Short-Answer Questions

**Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**
Kubernetes namespaces are only a logical separation for objects, RBAC, and quotas; they do not provide network isolation. Without `NetworkPolicy` enforcement, pods can communicate across namespaces by default, as shown by the `HTTP 200` result in Task 2. In a multi-tenant cloud, this is dangerous because a compromised workload in one namespace could probe or attack services in another tenant's namespace, causing a cross-tenant security breach.

**Q2. Explain the default-deny principle and how your NetworkPolicy implements it.**
Default-deny means access is blocked unless explicitly permitted, rather than allowed unless explicitly blocked — permit by exception, not by default.

The `default-deny-ingress` policy implements this by selecting all pods in `tenant-b` (`podSelector: {}`) and setting `policyTypes: [Ingress]` with no `ingress` rules defined. This means no traffic sources are explicitly allowed in.

As demonstrated in Task 4, this changes `tenant-b` from being reachable by any pod in the cluster to being reachable by none, until a specific allow-rule is added.

**Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**
| Feature | Containers | Virtual Machines |
|---|---|---|
| Isolation | Kernel namespaces + cgroups | Hypervisor + virtualized hardware |
| Kernel | Share host kernel | Each VM has its own kernel |
| Isolation strength | Lower | Stronger |
| Overhead | Lightweight | Higher |
| Main risk | Kernel vulnerability / container escape can affect other containers | Compromise is more isolated from other VMs |

**When to add a VM boundary?**

Use VMs when tenants are **mutually untrusted** and a cross-tenant breach could have a high impact, such as:
- Hosting competing customers
- Running untrusted third-party code


**Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?**
| Concept | Explanation |
|---|---|
| **Data remanence** | Residual data that remains on storage after deletion. Normal deletion may only remove the data pointer, leaving the underlying bytes recoverable. |
| **Cloud challenge** | Physical disks are usually virtualized, shared, and replicated, so users cannot directly overwrite or shred them. |
| **Cryptographic erasure** | Encrypt the data and destroy the encryption key when deletion is required. |
| **Why preferred** | Destroying the key makes all encrypted copies unreadable without needing physical access to the storage. |

**Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?**
| Task | Isolation Dimension | What It Tested |
|---|---|---|
| **Task 1** — Namespaces, Deployments | **Compute** | Separating tenant workloads |
| **Task 3** — ResourceQuota | **Compute** | Limiting shared resource consumption |
| **Task 2** — Cross-namespace probe | **Network** | Testing cross-tenant network access |
| **Task 4** — Default-deny NetworkPolicy | **Network** | Blocking cross-tenant network access |
| **Task 5** — Secrets + RBAC | **Storage** | Preventing unauthorized access to stored data |
| **Task 6** — Data remanence / secure wipe | **Storage** | Addressing residual data after deletion |

**Key Takeaway:**  
- **Compute:** Tasks 1 & 3  
- **Network:** Tasks 2 & 4  
- **Storage:** Tasks 5 & 6

---

## Cleanup & Teardown

```bash
# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab2
```
```bash
# Remove the Docker volume used for the remanence demo
docker volume rm ccse-vol
```

## Security Best-Practices Checklist

- [x] Tenants were separated into distinct namespaces (`tenant-a`, `tenant-b`) rather than sharing one namespace.
- [x] A default-deny `NetworkPolicy` blocks cross-tenant traffic, verified with a before (`HTTP 200`) / after (timeout) comparison.
- [x] A `ResourceQuota` prevents a noisy-neighbour tenant from exhausting shared cluster capacity.
- [x] Per-tenant Secrets are unreadable by other tenants, enforced by scoped RBAC (`Role` + `RoleBinding`) rather than trust in namespace naming.
- [x] Secure deletion / cryptographic erasure is understood as the practical answer to data remanence in cloud storage.

---

## Conclusion

This lab demonstrated that multi-tenant isolation on shared cloud infrastructure is not automatic — it must be deliberately engineered across compute, network, and storage dimensions. 

### Session A — Default Risks
- Separate namespaces did **not** prevent cross-tenant network access.
- Tenants could consume shared resources without limits.
- `ResourceQuota` was needed to control resource usage.

### Session B — Active Enforcement
- `NetworkPolicy` with **default-deny** blocked cross-tenant traffic.
- The previous `HTTP 200` request became a **timeout**, confirming network isolation.
- **RBAC + per-tenant ServiceAccounts** restricted access to secrets.
- The data remanence test showed that deleting a file does not guarantee secure deletion.
- **Cryptographic erasure** provides scalable and reliable data deletion in cloud environments.

### Key Takeaway
Secure multi-tenant isolation requires:
- **Compute:** Namespaces + ResourceQuota
- **Network:** Default-deny `NetworkPolicy`
- **Storage:** RBAC + secure data deletion

Controls should be **tested and verified**, not assumed to work just because they are configured.



