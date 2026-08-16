## Task 7 — Egress Default-Deny & Micro-Segmentation

The objective of this task is to enforce **default-deny egress** for `tenant-a` and then explicitly allow only the communication required for normal operation:

- DNS resolution to the cluster DNS service.
- TCP/80 access from `tenant-a` to the specific `app=web` workload in `tenant-b`.

This implements a **least-privilege egress allow-list** for the tenant.

| Evidence File | Purpose |
|---|---|
| `task7.1.png` | `tenant-b` web pod labels and Service verification showing `app=web` and TCP/80 |
| `task7.2.png` | `default-deny-egress` NetworkPolicy created and verified in `tenant-a` |
| `task7.3.png` | `allow-dns-egress` and `allow-tenant-b-web` policies verified in `tenant-a` |
| `task7.4.png` | Allowed `tenant-a` → `tenant-b` web-service probe returning `HTTP 200` |
| `task7.5.png` | Successful DNS resolution from `tenant-a` through the cluster DNS service |
| `task7.6.png` | Final `kubectl get networkpolicy -A` output showing the complete micro-segmentation configuration |

---

### Verify the Target Workload and Service

```bash
kubectl get pods -n tenant-b --show-labels
```

**Why:** The `app=web` label is required by the egress NetworkPolicy so that traffic is permitted only to the intended web workload in `tenant-b`.

**Result:**

```text
NAME                   READY   STATUS    RESTARTS   AGE   LABELS
web-7c56dcdb9b-sfdnm   1/1     Running   0          14h   app=web,pod-template-hash=7c56dcdb9b
```

The `tenant-b` web pod was running successfully and had the expected `app=web` label.

The corresponding Service was then verified:

```bash
kubectl get svc web -n tenant-b
```

**Why:** This confirms that the target web Service exists and exposes TCP port `80`.

**Result:**

```text
NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.207.129   <none>        80/TCP    14h
```

The `web` Service was available as a `ClusterIP` service at `10.96.207.129` on TCP/80.

**Evidence:**

<div align="left">
<img alt="tenant-b web pod and service verification" src="evidence lab2/task7.1.png">
</div>

---

### Apply Egress Default-Deny

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
    - Egress
EOF
```

**Why:** `podSelector: {}` selects all pods in `tenant-a`. With `policyTypes: [Egress]` and no egress rules, outbound traffic from these pods is denied by default.

This establishes the **default-deny egress baseline** before specific exceptions are added.

**Result:**

```text
networkpolicy.networking.k8s.io/default-deny-egress created
```

The policy was successfully created.

The policy was verified:

```bash
kubectl get networkpolicy -n tenant-a
```

**Result:**

```text
NAME                  POD-SELECTOR   AGE
default-deny-egress   <none>         14s
```

The `default-deny-egress` NetworkPolicy was present in `tenant-a`.

**Evidence:**

<div align="left">
<img alt="Default-deny egress policy applied" src="evidence lab2/task7.2.png">
</div>

---

### Allow DNS Egress

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF
```

**Why:** A default-deny egress policy would also block DNS unless it is explicitly permitted. This rule allows `tenant-a` pods to communicate with the cluster DNS pods in `kube-system` using UDP/TCP port `53`.

**Result:**

```text
networkpolicy.networking.k8s.io/allow-dns-egress created
```

The DNS egress exception was successfully created.

The policies in `tenant-a` were verified:

```bash
kubectl get networkpolicy -n tenant-a
```

**Result:**

```text
NAME                  POD-SELECTOR   AGE
allow-dns-egress      <none>         54s
default-deny-egress   <none>         90s
```

Both the default-deny baseline and DNS allow rule were present.

---

### Allow Only the Specific `tenant-b` Web Workload

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-tenant-b-web
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: tenant-b
          podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 80
EOF
```

**Why:** This creates a specific egress exception from `tenant-a` to only pods labelled `app=web` in the `tenant-b` namespace, and only on TCP port `80`.

This prevents `tenant-a` from having unrestricted access to other workloads or ports in `tenant-b`.

**Result:**

```text
networkpolicy.networking.k8s.io/allow-tenant-b-web created
```

The specific web-service egress exception was successfully created.

The policies in `tenant-a` were verified:

```bash
kubectl get networkpolicy -n tenant-a
```

**Result:**

```text
NAME                   POD-SELECTOR   AGE
allow-dns-egress       <none>         87s
allow-tenant-b-web     <none>         7s
default-deny-egress    <none>         2m3s
```

The `tenant-a` namespace now contained:

- `default-deny-egress`
- `allow-dns-egress`
- `allow-tenant-b-web`

**Evidence:**

<div align="left">
<img alt="tenant-a egress NetworkPolicies" src="evidence lab2/task7.3.png">
</div>

---

### Allow Corresponding Ingress to the Web Workload

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-tenant-a-to-web
  namespace: tenant-b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: tenant-a
      ports:
        - protocol: TCP
          port: 80
EOF
```

**Why:** `tenant-b` already has a `default-deny-ingress` policy. Therefore, the corresponding ingress permission is required so that traffic allowed by the `tenant-a` egress policy can actually reach the `app=web` workload.

The rule permits traffic from the `tenant-a` namespace to the `app=web` workload in `tenant-b` on TCP/80.

**Result:**

```text
networkpolicy.networking.k8s.io/allow-tenant-a-to-web created
```

The corresponding ingress exception was successfully created.

> **Note:** In the original terminal history, this command was not immediately followed by a `kubectl get networkpolicy -A` verification. The full cluster-wide verification is captured later, in the **Final NetworkPolicy Verification** section below.

---

### Validate Allowed Service Traffic

```bash
kubectl -n tenant-a run probe-allowed --rm -it \
  --image=curlimages/curl \
  --restart=Never \
  -- curl -s -m 5 http://10.96.207.129 \
  -o /dev/null \
  -w 'HTTP %{http_code}\n'
```

**Why:** This tests the exact communication path that was explicitly allowed by the NetworkPolicies:

`tenant-a → tenant-b/app=web → TCP/80`

**Result:**

```text
If you don't see a command prompt, try pressing enter.
HTTP 200
pod "probe-allowed" deleted
```

The test was repeated and produced the same result:

```text
HTTP 200
pod "probe-allowed" deleted
```

The `HTTP 200` response proves that the specifically permitted `tenant-a` → `tenant-b` web-service communication is working.

This is important because the goal is **not** to block all egress. The goal is to block egress by default while allowing only explicitly approved communication.

**Evidence:**

<div align="left">
<img alt="Allowed tenant-a to tenant-b web traffic returning HTTP 200" src="evidence lab2/task7.4.png">
</div>

---

### Validate DNS Traffic

```bash
kubectl -n tenant-a run dns-test --rm -it \
  --image=busybox:1.36 \
  --restart=Never \
  -- nslookup kubernetes.default.svc.cluster.local
```

**Why:** DNS is a required exception to the default-deny egress policy. This test verifies that `tenant-a` can still resolve Kubernetes service names through the cluster DNS service.

**Result:**

```text
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   kubernetes.default.svc.cluster.local
Address: 10.96.0.1

pod "dns-test" deleted
```

The DNS lookup succeeded.

The DNS server at `10.96.0.10` was reachable on port `53`, and `kubernetes.default.svc.cluster.local` successfully resolved to `10.96.0.1`.

This confirms that the explicit DNS egress exception is functioning correctly.

**Evidence:**

<div align="left">
<img alt="Successful DNS resolution from tenant-a" src="evidence lab2/task7.5.png">
</div>

---

### Final NetworkPolicy Verification

```bash
kubectl get networkpolicy -A
```

**Why:** This provides a cluster-wide verification of the NetworkPolicy objects involved in the micro-segmentation configuration.

**Result:**

```text
NAMESPACE   NAME                    POD-SELECTOR   AGE
tenant-a    allow-dns-egress        <none>         11m
tenant-a    allow-tenant-b-web      <none>         10m
tenant-a    default-deny-egress    <none>         12m
tenant-b    allow-tenant-a-to-web   app=web        8m26s
tenant-b    default-deny-ingress    <none>         15h
```

The final configuration confirms:

- `tenant-a` has a `default-deny-egress` policy.
- `tenant-a` has an explicit DNS egress allow rule.
- `tenant-a` has an explicit allow rule for the `tenant-b` `app=web` workload on TCP/80.
- `tenant-b` allows the corresponding ingress from `tenant-a` to `app=web`.
- `tenant-b` retains its `default-deny-ingress` policy.

**Evidence:**

<div align="left">
<img alt="Final NetworkPolicy configuration across namespaces" src="evidence lab2/task7.6.png">
</div>

---

### Task 7 Result

The egress configuration successfully implements **least-privilege micro-segmentation**.

The `default-deny-egress` policy establishes a deny-by-default baseline for all pods in `tenant-a`. Specific exceptions were then added only for the communication required by the workload.

| **Traffic** | **Result** |
| --- | --- |
| `tenant-a → DNS :53` | **ALLOWED** |
| `tenant-a → tenant-b/app=web :80` | **ALLOWED** |
| Other unspecified egress traffic | **DENIED by default** |

The `HTTP 200` result proves that the specifically permitted web service remains reachable.

The successful DNS lookup proves that required DNS functionality remains available.

Together, these results demonstrate **egress default-deny with explicit allow-list rules for DNS and a specific service**, providing namespace/workload-level micro-segmentation.

**Key Security Principle:**

> **Deny by default, permit only what is explicitly required.**

---