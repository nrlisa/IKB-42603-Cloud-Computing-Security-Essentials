# Lab 1: Cloud Account Security, Identity and Access Management
Course: IKB42603 Cloud Computing Security Essentials

Lab: Lab 1 

Topic: Identity governance, least privilege, LocalStack IAM and Kubernetes RBAC

Environment: LocalStack on localhost:4566 and kind Kubernetes cluster ccse-lab1

Name: Nurlisa Sofiya binti Mahadzir

Lab Summary // Objective

This lab demonstrates cloud identity management and access control using two different platforms:

LocalStack IAM was used to simulate AWS Identity and Access Management functions, including users, groups, policies and access keys.
Kubernetes RBAC was used to implement authorization control by defining roles and assigning permissions throug# Lab 1 — Cloud Account Security, Identity & Access Management

---
## Evidence Folder

All screenshots used for this report are stored in the `evidence lab1` folder.

| Evidence File | Purpose |
|---|---|
| `earlysetup.png` | `sts get-caller-identity` output confirming CLI is talking to LocalStack |
| `ev1task2.png` | Commands for creating the `Admins` group, attaching the admin policy, creating the admin user and adding it to the group |
| `ev1task2.1.png` | `Admins` group creation / policy attachment output |
| `ev1task2.2.png` | `CloudAdmin_Lisa` membership verification in `Admins` group |
| `ev1task3.1.png` | `Analyst_Raisha` read-only user creation output |
| `ev1task3.2.png` | `AmazonS3ReadOnlyAccess` policy attached to `Analyst_Raisha` |
| `task4.1.png` | Access key creation for `Analyst_Raisha` |
| `task4.2.png` | Access key listing and deactivation (rotation) for `Analyst_Raisha` |
| `setup1.png` | kind Kubernetes cluster creation |
| `setup2.png` | `kubectl cluster-info` / `kubectl get nodes` verification |
| `task5.png` | `dev` and `prod` namespace creation |
| `task6.1.png` | Service account `devsofia` creation |
| `task6.2.png` | Role `pod-reader` creation |
| `task6.3.png` | RoleBinding `dev-user-binding` creation |
| `task7.png` | RBAC `kubectl auth can-i` authorization test results |
| `verify.png` | RoleBinding YAML verification output |

## Overview

This lab is split into two sessions:

- **Session A (Week 1):** Cloud identity fundamentals using LocalStack (an AWS-compatible local simulator) — root user, IAM users, groups, policies, and access keys.
- **Session B (Week 2):** Real enforcement of access control using Kubernetes RBAC (Role-Based Access Control), plus a short audit/report.

Nothing in this lab touches a real cloud provider — LocalStack emulates AWS APIs on `localhost:4566`, and `kind` runs a throwaway Kubernetes cluster inside Docker on the local machine.

---
## Session A (Week 1) — Cloud Identity with LocalStack 
## Environment Setup

```bash
# 1. Confirm Docker is installed and running
docker --version
```
**Why:** Confirms Docker Engine is installed and the daemon is reachable before trying to launch any containers. If this fails, nothing else in the lab will work.

```bash
# 2. Start LocalStack (AWS-compatible cloud) in a container
docker run -d --name localstack -p 4566:4566 localstack/localstack
```
**Why:** Runs LocalStack in detached mode (`-d`), names the container `localstack` for easy reference, and maps container port 4566 to the host so the AWS CLI can reach it at `http://localhost:4566`. Port 4566 is LocalStack's single "edge" port that proxies to all emulated AWS services (IAM, S3, etc.).

```bash
# 3. Confirm it is healthy (should list running services)
curl http://localhost:4566/_localstack/health
```
**Why:** Hits LocalStack's built-in health-check endpoint to verify the emulator actually booted and which services are marked "running" before issuing IAM commands against it.

```bash
# Configure dummy credentials (LocalStack accepts any value)
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```
**Why:** The AWS CLI refuses to run without *some* credentials configured, even against a local emulator. LocalStack doesn't validate these values, so any string works — this just satisfies the CLI's requirement.

```bash
# Test: this talks to LocalStack, NOT real AWS
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```
**Why:** `--endpoint-url` redirects the AWS CLI away from real AWS servers and toward the local LocalStack container. `sts get-caller-identity` returns the identity (account/user ARN) the CLI is currently operating as   identity subsequent commands run under.

**Output:**
```json
{
    "UserId": "000000000000",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```
The account ID `000000000000` confirms the commands were executed against LocalStack, not real AWS.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/earlysetup.png">

---

## Task 1 — Map the Cloud Identity Landscape

| Concept | AWS term | Purpose |
|---|---|---|
| All-powerful owner | **Root user** | The account owner identity created at sign-up, with unrestricted access to every resource and setting. It should be locked away (MFA-protected, credentials never used day-to-day) and reserved only for account-level tasks (e.g. closing the account, changing billing) because a compromise gives an attacker total control. |
| Human/app identity | **IAM User** | A persistent identity representing a specific person or application that needs to interact with the cloud account. Each user gets its own credentials and permissions, so access can be granted, audited, and revoked individually instead of everyone sharing the root login. |
| Permission bundle | **IAM Policy** | A JSON document that explicitly defines what actions are *allowed* or *denied* on which resources. Policies are the actual enforcement rules — attaching a policy is what grants (or restricts) capability to a user, group, or role. |
| Collection of users | **IAM Group** | A container of IAM users that lets you attach a policy once and have it apply to every member. Groups make permission management scalable — update the group's policy and every user in it is updated at the same time, instead of editing each user individually. |
| Temporary identity | **IAM Role** | An identity with no long-term credentials of its own; it is *assumed* temporarily by a user, application, or service to gain a specific set of permissions for a limited time/session. Roles avoid the risk of long-lived access keys by issuing short-term, expiring credentials. |

---

## Task 2 — Create a Least-Privilege Admin (Stop Using Root)

```bash
EP='--endpoint-url=http://localhost:4566'
```
**Why:** Stores the LocalStack endpoint flag in a variable so it doesn't have to be retyped on every command — pure convenience/readability.

```bash
# 2.1 Create a group and attach an admin policy to the GROUP
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
    --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```
**Result:** The group Admins was created successfully.
**Why:** Creates an `Admins` group and attaches AWS's managed `AdministratorAccess` policy **to the group itself**, not to any individual user. This is the core best practice: permissions live on the group; users simply join or leave it.

Verification of the attached policy:

```bash
aws $EP iam list-attached-group-policies --group-name Admins
```

**Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AdministratorAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AdministratorAccess"
        }
    ]
}
```
This confirms `AdministratorAccess` is attached to the `Admins` group.

```bash
# 2.2 Create your personal admin user (replace YOURNAME)
aws $EP iam create-user --user-name CloudAdmin_Lisa
```
**Why:** Creates a dedicated, named IAM user to replace root for daily administrative work. Using a personal, named identity (instead of root) means actions are attributable to a specific person and root credentials never need to be exposed.
**Result:** The user `CloudAdmin_Lisa` was created successfully.

```bash
# 2.3 Put the user in the group (permissions flow from the group)
aws $EP iam add-user-to-group --group-name Admins \
    --user-name CloudAdmin_Lisa
```
**Result:** The group Admins was created successfully.
**Why:** Membership in `Admins` is what actually grants `CloudAdmin_Lisa` the AdministratorAccess permissions — the user itself has no policy attached directly.

```bash
# 2.4 Verify the membership
aws $EP iam get-group --group-name Admins
```
**Why:** Confirms the user was correctly added to the group and lists the group's attached policy, providing evidence the setup worked as intended.

**Output (summary):**
```json
{
    "Users": [
        {
            "UserName": "CloudAdmin_Lisa",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_Lisa"
        }
    ],
    "Group": {
        "GroupName": "Admins",
        "Arn": "arn:aws:iam::000000000000:group/Admins"
    }
}
```
This proves `CloudAdmin_Lisa` is a member of `Admins`, with the admin permission inherited from the group rather than attached directly to the user.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/ev1task2.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/ev1task2.1.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/ev1task2.2.png">
---

## Task 3 — Enforce Least Privilege with a Scoped Policy

```bash
# 3.1 Create a read-only user
aws $EP iam create-user --user-name Analyst_Raisha
```
**Why:** Creates a separate identity for a teammate who only needs to *view* data, never modify it — the first step in giving them the minimum access necessary for their role.
**Result:** The user `Analyst_Raisha` was created successfully.

```bash
# 3.2 Attach a scoped, read-only policy (S3 read only)
aws $EP iam attach-user-policy --user-name Analyst_Raisha \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```
**Why:** Directly attaches AWS's managed `AmazonS3ReadOnlyAccess` policy, which permits only `Get`/`List`-type S3 actions and explicitly excludes write/delete actions. This demonstrates fine-grained authorization — the user can read but cannot modify data.

```bash
# 3.3 List what the user can do
aws $EP iam list-attached-user-policies --user-name Analyst_Raisha
```
**Why:** Verifies exactly which policy is attached to `Analyst_Raisha`, proving it has read-only access and nothing broader.

**Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```
This proves that `Analyst_Raisha` only has the `AmazonS3ReadOnlyAccess` policy attached, and nothing broader.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/ev1task3.1.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/ev1task3.2.png">

---

## Task 4 — Credential Hygiene & Access Keys

```bash
# 4.1 Create an access key for the Analyst
aws $EP iam create-access-key --user-name Analyst_Raisha
```
**Why:** Generates a programmatic Access Key ID + Secret Access Key pair so the Analyst identity could authenticate via CLI/SDK rather than a console password. This output must be saved immediately — the secret key is not retrievable again later.
**Result:** An access key was created for `Analyst_Raisha`.


```bash
# 4.2 List access keys (note the AccessKeyId and status)
aws $EP iam list-access-keys --user-name Analyst_Raisha

```
**Why:** Confirms the key was created and shows its `AccessKeyId` and `Status` (Active/Inactive), which is needed to reference the key in the next step.

**Output:**
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_Raisha",
            "AccessKeyId": "<PASTE_KEY_ID>",
            "Status": "Active",
            "CreateDate": "2026-07-29T05:29:06.789002+00:00"
        }
    ]
}
```

```bash
# 4.3 Rotate: deactivate the old key (paste the AccessKeyId)
aws $EP iam update-access-key --user-name Analyst_Raisha \
    --access-key-id <PASTE_KEY_ID> --status Inactive
```
**Result:** The access key status is now `Inactive`, which demonstrates key rotation/deactivation.
Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task4.1.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task4.2.png">

**Why:** Simulates key rotation by disabling the key without deleting it outright. In practice, rotation means: issue a new key, update systems to use it, then deactivate/delete the old one — this limits how long any single long-lived credential remains valid and usable if leaked.

*End of Session A.*

---

## Session B (Week 2) — Enforced Access Control with Kubernetes RBAC

LocalStack demonstrates IAM *mechanics* but doesn't fully enforce them. Kubernetes RBAC is the real enforcement engine — it will actually block disallowed actions.

```bash
# Create a throwaway cluster (runs inside Docker)
kind create cluster --name ccse-lab1
```
**Why:** `kind` (Kubernetes-in-Docker) spins up a real, disposable Kubernetes cluster entirely inside Docker containers on the local machine — no cloud account or cost required.

```bash
# Confirm it is up
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```
**Why:** Verifies the cluster's control plane is reachable and lists its node(s), confirming the cluster is ready before creating any resources in it.
**Result:** The local kind cluster `ccse-lab1` was created successfully, and kubectl was configured to use context `kind-ccse-lab1`.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/setup1.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/setup2.png">
---

## Task 5 — Separate Environments with Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```
**Why:** Namespaces partition a single cluster into isolated logical environments. Creating separate `dev` and `prod` namespaces lets us later prove that permissions granted in one namespace do *not* automatically extend to another — a core test of least privilege and blast-radius containment within one cluster.
**Result:** The namespaces `dev` and `prod` were created and listed as `Active`.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task5.png">

---

## Task 6 — Define a Role and Bind It (Least Privilege)

```bash
# 6.1 Create a service account to represent a developer
kubectl create serviceaccount devsofia -n dev
```
**Why:** A ServiceAccount is a non-human identity Kubernetes uses to authenticate workloads/API calls (here representing "a developer"), scoped to the `dev` namespace.
**Result:** The service account `devsofia` was created in the `dev` namespace.

```bash
# 6.2 Create a Role that allows only get/list/watch on pods in dev
kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods
```
**Why:** A `Role` is a namespaced set of permissions (verbs on resources). This one grants only read-type actions (`get`, `list`, `watch`) on `pods`, and only within the `dev` namespace — explicitly excluding `delete`, `create`, `update`, etc.
**Result:** The Role `pod-reader` was created in the `dev` namespace. It allows only `get`, `list`, and `watch` actions on pods.

```bash
# 6.3 Bind the Role to the service account
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:devsofia

```
**Why:** A `Role` alone grants nothing until it is *bound* to an identity. The `RoleBinding` links the `pod-reader` Role to the `devsofia` ServiceAccount, so that identity now actually has those permissions — and only within the `dev` namespace, since RoleBindings (unlike ClusterRoleBindings) are namespace-scoped.
**Result:** The RoleBinding `dev-user-binding` binds the `pod-reader` Role to the `devsofia` service account.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task6.1.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task6.2.png">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task6.3.png">
---

## Task 7 — Test That Access Control Works

```bash
SA=system:serviceaccount:dev:dev-user

# Should be YES — reading pods in dev is allowed
kubectl auth can-i list pods -n dev --as=$SA

# Should be NO — deleting pods is not granted
kubectl auth can-i delete pods -n dev --as=$SA

# Should be NO — the role does not extend to prod
kubectl auth can-i list pods -n prod --as=$SA
```
**Why:** `kubectl auth can-i --as=<identity>` simulates what a given identity is authorized to do, without needing to actually attempt the action. Testing all three cases proves the RBAC boundary works exactly as configured: allowed action succeeds, disallowed verb (delete) is blocked, and cross-namespace access (prod) is blocked — even though it's the same cluster.

**Results:**
- `list pods -n dev` → `yes` — allowed because `pod-reader` grants `list` on pods in `dev`.
- `delete pods -n dev` → `no` — blocked because the Role only grants `get`, `list`, and `watch`; delete was never granted.
- `list pods -n prod` → `no` — blocked because the Role and RoleBinding are namespaced to `dev` and do not extend to `prod`.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/task7.png">

**Report answer — Authentication vs. Authorization:**
In all three checks, the service account is successfully **authenticated** — Kubernetes recognizes `system:serviceaccount:dev:dev-user` as a valid, known identity in every case; that step never fails. What differs is **authorization**: for `list pods -n dev`, the RBAC Role+RoleBinding explicitly grants that verb/resource/namespace combination, so it's authorized (YES). For `delete pods -n dev` and `list pods -n prod`, the identity is still authenticated, but the RBAC rules attached to it never grant a `delete` verb, nor any permissions in the `prod` namespace — so the **authorization** step is what blocks the request (NO/NO), not authentication. This demonstrates the RBAC principle of least privilege: identity is verified, but access is separately and precisely scoped.

---

## Verification Command

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```
**Why:** Dumps the full RoleBinding definition as YAML — proof that the binding (subject, role reference, namespace) exists exactly as configured, submitted as evidence RBAC is correctly in place.

**Output:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-07-29T05:48:38Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "701"
  uid: 91124053-fdc5-418a-a916-ec078374971c
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: devsofia
  namespace: dev
```
This confirms that the `dev-user-binding` RoleBinding connects the `devsofia` service account to the `pod-reader` Role in the `dev` namespace.

Evidence: <div align="left">
<img alt="Screenshot 2026-07-29 115703" src="evidence lab1/verify.png">
---

## Short-Answer Questions

**Q1. Why is attaching policies to groups better than attaching them directly to users?**
Attaching policies to a group centralizes permission management — updating the group's policy instantly updates every member, and adding/removing a user from the group instantly grants/revokes access. Attaching policies per-user instead means every permission change has to be repeated individually for each user, which doesn't scale and is much easier to get inconsistent or forget to update, increasing audit difficulty and risk.

**Q2. What is the difference between an IAM User and an IAM Role?**
An IAM User is a persistent identity with its own long-term credentials (password and/or access keys) representing a specific person or application. An IAM Role has no credentials of its own and cannot be logged into directly — it is *assumed* temporarily by a trusted user, application, or service, which is then issued short-lived, expiring credentials for the duration of that session. Roles are safer for many use cases because there's no long-lived secret to leak.

**Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.**
The Analyst account was given only `AmazonS3ReadOnlyAccess` — exactly the permission needed for its job (viewing data) and nothing more. This is least privilege: granting the minimum access required to perform a task. If the Analyst's credentials are compromised, the attacker inherits only read access to S3; they cannot delete, modify, or create resources, and have zero access to any other AWS service. This caps the maximum possible damage (the "blast radius") of that one compromised identity, in sharp contrast to a compromised admin account, which could cause unlimited damage across the entire account.

**Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?**
A `Role` defines *what* permissions exist — a set of verbs (get, list, delete, etc.) allowed on specific resources within a namespace. It does not grant anything to anyone by itself. A `RoleBinding` defines *who* gets those permissions — it links (binds) a Role to one or more subjects (users, groups, or service accounts). Permissions only take effect once a Role is bound to a subject via a RoleBinding.

**Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?**
The `dev-user` service account failed to access `prod` because its RoleBinding (`dev-user-binding`) only binds the `pod-reader` Role within the `dev` namespace — namespaced RoleBindings do not extend to other namespaces. There is no Role or RoleBinding for `dev-user` in `prod` at all, so the request is denied by default. This demonstrates the **principle of least privilege** (and, more specifically, environment/namespace isolation) — an identity should only be granted access to the exact resources and environments it needs, and Kubernetes defaults to deny-all unless a rule explicitly grants access.

---

## Cleanup & Teardown

```bash
# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab1
```
```bash
# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```
## Security Best-Practices Checklist

- [x] Root user was not used for daily administrative tasks — a dedicated identity, `CloudAdmin_Lisa`, was created instead.
- [x] Admin permissions were granted through the `Admins` group rather than being attached directly to the admin user, so access can be managed centrally.
- [x] A least-privilege identity, `Analyst_Raisha`, was created and scoped only to `AmazonS3ReadOnlyAccess`.
- [x] Access keys were created, listed, and deactivated to demonstrate credential rotation and hygiene.
- [x] Kubernetes RBAC correctly enforced boundaries — deleting pods in `dev` and listing pods in `prod` were both blocked for the `devsofia` service account.
- [x] Environments were isolated using separate `dev` and `prod` namespaces, preventing permissions from unintentionally crossing over.

---

## Conclusion

This lab demonstrated how identity and access management principles apply across two different environments — a simulated cloud platform and a real orchestration system. In LocalStack, administrative access was managed responsibly by attaching the `AdministratorAccess` policy to a group rather than an individual user, keeping root credentials out of daily use. A separate, scoped-down identity (`Analyst_Raisha`) showed how least privilege limits the impact of a compromised account to a single, low-risk permission set.

In Kubernetes, RBAC proved to be a stricter, actively enforced form of access control compared to LocalStack IAM's simulation. The `devsofia` service account could only perform the exact actions defined by its Role — read-only access to pods, and only within the `dev` namespace. Attempts to delete pods or reach into `prod` were denied by default, confirming that Kubernetes authorization does not extend permissions beyond what is explicitly granted.

Overall, the lab reinforced that strong identity governance isn't just about creating accounts — it's about deliberately scoping what each identity can do, grouping permissions for easier management, and verifying that those boundaries actually hold up when tested.
