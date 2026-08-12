# Lab 2: Secure Isolation & Multi-Tenancy
Course: IKB42603 Cloud Computing Security Essentials

Lab: Lab 2

Topic: Compute, network and storage isolation, default-deny NetworkPolicy, RBAC secret isolation, data remanence

Environment: kind Kubernetes cluster ccse-lab2 (Calico CNI) and Docker

Name: Nurlisa Sofiya binti Mahadzir

## Lab Summary // Objective

This lab demonstrates multi-tenant isolation on shared cloud infrastructure using Kubernetes and Docker:

Compute isolation was modelled by placing two simulated tenants — `tenant-a` and `tenant-b` — into separate namespaces on the same cluster, and containing resource usage with a ResourceQuota.
Network isolation was enforced with a default-deny Kubernetes NetworkPolicy, proving that cross-tenant traffic which succeeded by default was blocked once the policy was applied.
Storage isolation was enforced with Kubernetes RBAC, proving that a service account scoped to one tenant cannot read another tenant's Secret.
Data remanence was explored using a Docker volume, showing that a normally deleted file can still leave recoverable bytes on disk, and that a secure "wipe before delete" reduces this risk.

---

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

Session A shows the problem: on shared infrastructure, isolation is not automatic. Session B applies the controls — network policy and RBAC — that make the same infrastructure safely separated. The cluster uses Calico as its CNI (with the default kind CNI disabled) because the default kind network does not enforce `NetworkPolicy` objects; without Calico, the network isolation task would silently do nothing.

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
**Why:** The stock `kind` cluster ships with a basic CNI that does not enforce `NetworkPolicy` resources — applying a policy on it would have no real effect. Disabling the default CNI and installing Calico gives the cluster a CNI that actually reads and enforces `NetworkPolicy` objects, which is required for Task 4 to demonstrate real network isolation rather than a policy that is created but silently ignored.
**Result:** The cluster `ccse-lab2` was created successfully, Calico was installed, and the `calico-node` DaemonSet reported a successful rollout, confirming policy enforcement is active before any tenant workloads were created.

Evidence: <div align="left">
<img alt="kind cluster creation with Calico CNI" src="evidence lab2/setup1.png">
<img alt="Calico rollout status" src="evidence lab2/setup2.png">

---

## Task 1 — Two Tenants on One Cluster

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```
**Why:** Namespaces are used here to model two separate customers (tenants) sharing the same physical cluster — the same pattern used for `dev`/`prod` isolation in Lab 1, now applied to a multi-tenancy scenario.
**Result:** The namespaces `tenant-a` and `tenant-b` were created successfully.

```bash
# Deploy a simple web server for each tenant
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
```
**Why:** Gives each tenant a running workload and a ClusterIP Service, so that cross-tenant reachability can be tested in Task 2. Using identical `nginx` deployments in both namespaces isolates the variable being tested to *namespace boundary*, not application differences.
**Result:** A `web` Deployment and Service were created in both `tenant-a` and `tenant-b`.

```bash
kubectl get pods,svc -n tenant-a
```
**Why:** Confirms the pod is `Running` and the Service has a ClusterIP allocated before attempting to reach it from the other tenant.

Evidence: <div align="left">
<img alt="Namespace creation" src="evidence lab2/task1.png">

---

## Task 2 — Observe the Default-Open Risk

```bash
# Get tenant-b's service IP
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo

# From tenant-a, curl tenant-b (replace <B_IP>)
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
    -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```
**Why:** By default, Kubernetes does not isolate pods from each other at the network layer — any pod in the cluster can route to any other pod's IP or Service, regardless of namespace, unless a `NetworkPolicy` says otherwise. Launching a throwaway `curl` pod inside `tenant-a` and pointing it at `tenant-b`'s Service tests this directly.
**Result:** The probe returned `HTTP 200`, proving `tenant-a` successfully reached `tenant-b`'s web server across the namespace boundary with no policy in place. This is the default-open risk: namespaces alone provide *logical* organization, not network isolation, so on shared infrastructure a compromised or malicious workload in one tenant's namespace could freely reach another tenant's services unless network policy is explicitly configured.

Evidence: <div align="left">
<img alt="tenant-b ClusterIP lookup" src="evidence lab2/task2.png">

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
**Why:** Isolation is not only about network reachability — it also covers shared *capacity*. Without a quota, one tenant could request unbounded CPU, memory, or pod count and starve every other tenant on the same node(s) (a "noisy neighbour" problem). Applying a `ResourceQuota` caps what `tenant-a` can consume, regardless of intent (misconfiguration or abuse).
**Result:** `tenant-a-quota` was created in the `tenant-a` namespace, capping requested CPU to `1`, memory to `512Mi`, and pod count to `5`. `kubectl describe` confirmed these hard limits are now enforced by the API server for any further object creation in that namespace.

Evidence: <div align="left">
<img alt="ResourceQuota applied" src="evidence lab2/task3.png">

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
**Why:** An empty `podSelector: {}` matches every pod in the `tenant-b` namespace, and `policyTypes: [Ingress]` with no `ingress` rules means no traffic is allowed in from anywhere — this is the "deny by default, permit by exception" segmentation principle. Re-running the *exact same* probe from Task 2 isolates the NetworkPolicy as the only variable that changed, giving a clean before/after comparison.
**Result:** The same probe that previously returned `HTTP 200` now timed out after 5 seconds with no response, proving the default-deny `NetworkPolicy` blocked `tenant-a`'s traffic into `tenant-b`. This is the enforced fix for the default-open risk observed in Task 2 — namespace separation plus a `NetworkPolicy` together provide real network isolation, whereas namespaces alone did not.

Evidence: <div align="left">
<img alt="Default-deny NetworkPolicy applied" src="evidence lab2/task4.1.png">
<img alt="Probe timing out after policy applied" src="evidence lab2/task4.2.png">

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
**Why:** Each tenant's Secret represents sensitive tenant data (e.g. an API key or credential). The `app-a` service account is scoped with a `Role` + `RoleBinding` (same least-privilege pattern as Lab 1) that only grants `get` on `secrets` inside `tenant-a`. Testing `can-i` against both namespaces proves storage isolation is enforced by RBAC, not just by namespace naming convention.
**Result:**
- `get secrets -n tenant-a` → `yes` — allowed because the `reader` Role and its RoleBinding grant `app-a` that permission inside its own namespace.
- `get secrets -n tenant-b` → `no` — blocked because `app-a`'s RoleBinding does not exist in, and is not bound to, `tenant-b`; RBAC defaults to deny.

This confirms `tenant-a`'s workload identity cannot read `tenant-b`'s secret data even though both secrets live in the same physical cluster.

Evidence: <div align="left">
<img alt="Secrets created per tenant" src="evidence lab2/task5.1.png">
<img alt="Service account, role and rolebinding creation" src="evidence lab2/task5.2.png">
---

## Task 6 — Data Remanence & Secure Deletion

```bash
# Create a file, delete it normally, then show the bytes may persist
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```
**Why:** A normal `rm` only removes the filesystem's pointer (directory entry) to the data — it does not necessarily overwrite the underlying disk blocks. Scanning the raw volume for the string afterward tests whether the "deleted" content is still physically recoverable, which is the definition of data remanence.
**Result:** The `grep` scan located the string `SENSITIVE-PATIENT-RECORD` in the volume even after `rm` had been run and the scan completed, demonstrating that normal deletion left recoverable bytes on disk — a real risk if that volume/disk were later reused, resold, or accessed by another tenant.

```bash
# Secure wipe: overwrite before delete (shred)
docker run --rm -v ccse-vol:/data alpine sh -c \
'echo SENSITIVE > /data/phi2.txt; sync; \
dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; \
echo wiped'
```
**Why:** `dd` overwrites the file's data blocks with zero bytes *before* the file is unlinked, so no readable content remains at that location for a subsequent scan to recover. This models the "shred before delete" mitigation for physical media where direct block access is possible.
**Result:** The file's content was overwritten with zeros prior to deletion, mitigating recovery of the sensitive string from that file's blocks. In real cloud storage, the operator rarely has this level of control over physical blocks (multi-tenant, virtualized, replicated storage), so the practical equivalent used in production is **cryptographic erasure** — encrypting data at rest and destroying the encryption key, which instantly renders all copies of the data unreadable without needing physical block access.

Evidence: <div align="left">
<img alt="Data remanence scan showing recoverable content" src="evidence lab2/task6.png">
---

## Verification Command

```bash
kubectl get networkpolicy -A
kubectl describe resourcequota tenant-a-quota -n tenant-a
```
**Why:** Dumps the cluster-wide list of `NetworkPolicy` objects and the current status of the `ResourceQuota`, providing evidence that both the network-isolation and compute-isolation controls are still correctly in place at the end of the lab.
**Result:** `default-deny-ingress` was confirmed present in `tenant-b`, and `tenant-a-quota` showed the configured hard limits (`requests.cpu: 1`, `requests.memory: 512Mi`, `pods: 5`) with current usage beneath them.

Evidence: <div align="left">
<img alt="NetworkPolicy and ResourceQuota verification" src="evidence lab2/verify.png">

---

## Short-Answer Questions

**Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in multi-tenant cloud?**
Kubernetes namespaces are a *logical* partition for organizing objects (names, RBAC scope, quotas) — they do not create a network boundary on their own. Unless a CNI that enforces `NetworkPolicy` is installed and a policy is applied, every pod IP is routable from every other pod in the cluster regardless of namespace, which is what Task 2 demonstrated with the `HTTP 200` result. In a multi-tenant cloud, this is dangerous because a single compromised or malicious workload in one tenant's namespace can freely probe, connect to, or attack another tenant's services, turning what should be a contained incident into a cross-tenant breach.

**Q2. Explain the default-deny principle and how your NetworkPolicy implements it.**
Default-deny means access is blocked unless explicitly permitted, rather than allowed unless explicitly blocked — permit by exception, not by default. The `default-deny-ingress` policy implements this by selecting all pods in `tenant-b` (`podSelector: {}`) and setting `policyTypes: [Ingress]` with no `ingress` rules defined, meaning no traffic sources are ever explicitly allowed in. As demonstrated in Task 4, this flips `tenant-b` from being reachable by any pod in the cluster to being reachable by none, until a specific allow-rule is added.

**Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?**
Containers share the host machine's kernel and are isolated from each other using kernel namespaces and cgroups, which is lighter-weight but means a kernel-level vulnerability or container-escape exploit can potentially affect every container on that host. Virtual machines are isolated by a hypervisor that virtualizes hardware and gives each VM its own kernel, which is a stronger, more resource-heavy isolation boundary because a compromise inside one VM does not directly expose the kernels of other VMs. A VM boundary should be added when tenants are mutually untrusted and the cost of a cross-tenant breach is high (e.g. hosting workloads for competing customers, or running third-party/untrusted code), since the stronger hardware-level isolation outweighs the extra overhead.

**Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?**
Data remanence is the residual representation of data that remains on storage media after it has been "deleted," because normal deletion typically just removes the pointer to the data rather than overwriting the underlying bytes — as shown in Task 6, where the deleted string was still recoverable by scanning the raw volume. In cloud environments, the operator usually does not have direct access to the physical disks (which are virtualized, shared, and replicated across a provider's infrastructure), so block-level overwriting/shredding is often impractical or impossible. Cryptographic erasure solves this by encrypting the data at rest and simply destroying the encryption key when deletion is required — instantly making every copy of the data unreadable everywhere it was replicated, without needing physical access to any disk.

**Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?**
- Task 1 (namespaces, deployments) and Task 3 (ResourceQuota) — **compute isolation**: separating tenant workloads and capping shared resource consumption.
- Task 2 (cross-namespace probe) and Task 4 (default-deny NetworkPolicy) — **network isolation**: exposing, then closing, cross-tenant network reachability.
- Task 5 (per-tenant Secrets + RBAC) and Task 6 (data remanence/secure wipe) — **storage isolation**: preventing cross-tenant access to stored data and addressing what remains after deletion.

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

This lab demonstrated that multi-tenant isolation on shared cloud infrastructure is not automatic — it must be deliberately engineered across compute, network, and storage dimensions. Session A exposed the default-open risk: two tenants placed in separate namespaces could still freely reach each other's services over the network, and nothing prevented one tenant from consuming unbounded shared resources until a `ResourceQuota` was applied.

Session B closed these gaps with active enforcement. A default-deny `NetworkPolicy` turned the same cross-tenant request that previously succeeded (`HTTP 200`) into a timeout, proving network isolation only exists once explicitly configured. RBAC scoped to a per-tenant service account proved that storage/secret isolation likewise depends on explicit permission grants, not on namespace boundaries alone. Finally, the data remanence exercise showed that "deleting" a file does not guarantee the data is gone, reinforcing why cloud providers and tenants rely on cryptographic erasure rather than physical overwriting for reliable, scalable secure deletion.

Overall, the lab reinforced that secure isolation in a shared cloud environment requires the same three controls working together: compute boundaries and quotas, default-deny network segmentation, and RBAC-enforced storage access — each independently verified by testing that the control actually blocks the disallowed action, not merely assumed from configuration intent.