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

## Overview

This lab is split into two sessions:

- **Session A (Week 1):** Cloud identity fundamentals using LocalStack (an AWS-compatible local simulator) — root user, IAM users, groups, policies, and access keys.
- **Session B (Week 2):** Real enforcement of access control using Kubernetes RBAC (Role-Based Access Control), plus a short audit/report.

Nothing in this lab touches a real cloud provider — LocalStack emulates AWS APIs on `localhost:4566`, and `kind` runs a throwaway Kubernetes cluster inside Docker on the local machine.

---

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
**Why:** `--endpoint-url` redirects the AWS CLI away from real AWS servers and toward the local LocalStack container. `sts get-caller-identity` returns the identity (account/user ARN) the CLI is currently operating as — this is the first required deliverable screenshot, proving which identity subsequent commands run under.

📸 **Screenshot 1:** Output of `sts get-caller-identity`

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
**Why:** Creates an `Admins` group and attaches AWS's managed `AdministratorAccess` policy **to the group itself**, not to any individual user. This is the core best practice: permissions live on the group; users simply join or leave it.

```bash
# 2.2 Create your personal admin user (replace YOURNAME)
aws $EP iam create-user --user-name CloudAdmin_YOURNAME
```
**Why:** Creates a dedicated, named IAM user to replace root for daily administrative work. Using a personal, named identity (instead of root) means actions are attributable to a specific person and root credentials never need to be exposed.

```bash
# 2.3 Put the user in the group (permissions flow from the group)
aws $EP iam add-user-to-group --group-name Admins \
    --user-name CloudAdmin_YOURNAME
```
**Why:** Membership in `Admins` is what actually grants `CloudAdmin_YOURNAME` the AdministratorAccess permissions — the user itself has no policy attached directly.

```bash
# 2.4 Verify the membership
aws $EP iam get-group --group-name Admins
```
**Why:** Confirms the user was correctly added to the group and lists the group's attached policy, providing evidence the setup worked as intended.

📸 **Screenshot 2:** `get-group Admins` output showing `CloudAdmin_YOURNAME` as a member.

---

## Task 3 — Enforce Least Privilege with a Scoped Policy

```bash
# 3.1 Create a read-only user
aws $EP iam create-user --user-name Analyst_YOURNAME
```
**Why:** Creates a separate identity for a teammate who only needs to *view* data, never modify it — the first step in giving them the minimum access necessary for their role.

```bash
# 3.2 Attach a scoped, read-only policy (S3 read only)
aws $EP iam attach-user-policy --user-name Analyst_YOURNAME \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```
**Why:** Directly attaches AWS's managed `AmazonS3ReadOnlyAccess` policy, which permits only `Get`/`List`-type S3 actions and explicitly excludes write/delete actions. This demonstrates fine-grained authorization — the user can read but cannot modify data.

```bash
# 3.3 List what the user can do
aws $EP iam list-attached-user-policies --user-name Analyst_YOURNAME
```
**Why:** Verifies exactly which policy is attached to `Analyst_YOURNAME`, proving it has read-only access and nothing broader.

📸 **Screenshot 3:** `list-attached-user-policies` for the Analyst showing only the read-only policy.

**Report answer — Blast radius:**
If the Analyst account's credentials were stolen, the damage is limited because the account can only *read* S3 data — it has no permission to create, modify, delete, or exfiltrate-by-altering any resource, and no permissions outside S3 at all. An attacker with these credentials could view data but couldn't destroy it, plant backdoors, spin up new resources, or pivot into other services. Compare this to a stolen `CloudAdmin` (AdministratorAccess) account, where the attacker could do *anything* — delete all resources, create new IAM users for persistence, exfiltrate everything, or run up billing. This difference in "how much damage is possible from this one compromised identity" is exactly what **blast-radius reduction** means: scoping permissions tightly shrinks the worst-case outcome of any single credential leak.

---

## Task 4 — Credential Hygiene & Access Keys

```bash
# 4.1 Create an access key for the Analyst
aws $EP iam create-access-key --user-name Analyst_YOURNAME
```
**Why:** Generates a programmatic Access Key ID + Secret Access Key pair so the Analyst identity could authenticate via CLI/SDK rather than a console password. This output must be saved immediately — the secret key is not retrievable again later.

```bash
# 4.2 List access keys (note the AccessKeyId and status)
aws $EP iam list-access-keys --user-name Analyst_YOURNAME

📸 **Screenshot 4:** Output of `list-access-keys` showing the created key and its status.
```
**Why:** Confirms the key was created and shows its `AccessKeyId` and `Status` (Active/Inactive), which is needed to reference the key in the next step.

```bash
# 4.3 Rotate: deactivate the old key (paste the AccessKeyId)
aws $EP iam update-access-key --user-name Analyst_YOURNAME \
    --access-key-id <PASTE_KEY_ID> --status Inactive
```
**Why:** Simulates key rotation by disabling the key without deleting it outright. In practice, rotation means: issue a new key, update systems to use it, then deactivate/delete the old one — this limits how long any single long-lived credential remains valid and usable if leaked.

**Report note:** Long-lived access keys are risky because, unlike role-based temporary credentials, they don't expire on their own — if leaked (e.g. committed to a public repo), they remain usable indefinitely until someone notices and manually deactivates them. This is why AWS best practice prefers short-lived roles over standing access keys wherever possible.

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

---

## Task 5 — Separate Environments with Namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```
**Why:** Namespaces partition a single cluster into isolated logical environments. Creating separate `dev` and `prod` namespaces lets us later prove that permissions granted in one namespace do *not* automatically extend to another — a core test of least privilege and blast-radius containment within one cluster.

📸 **Screenshot 5:** Output of `kubectl get namespaces` showing both namespaces.

---

## Task 6 — Define a Role and Bind It (Least Privilege)

```bash
# 6.1 Create a service account to represent a developer
kubectl create serviceaccount dev-user -n dev
```
**Why:** A ServiceAccount is a non-human identity Kubernetes uses to authenticate workloads/API calls (here representing "a developer"), scoped to the `dev` namespace.

```bash
# 6.2 Create a Role that allows only get/list/watch on pods in dev
kubectl create role pod-reader -n dev \
    --verb=get,list,watch --resource=pods
```
**Why:** A `Role` is a namespaced set of permissions (verbs on resources). This one grants only read-type actions (`get`, `list`, `watch`) on `pods`, and only within the `dev` namespace — explicitly excluding `delete`, `create`, `update`, etc.

```bash
# 6.3 Bind the Role to the service account
kubectl create rolebinding dev-user-binding -n dev \
    --role=pod-reader --serviceaccount=dev:dev-user

📸 **Screenshot 6:** Output of `kubectl get rolebinding dev-user-binding -n dev -o yaml` showing the created RoleBinding.
```
**Why:** A `Role` alone grants nothing until it is *bound* to an identity. The `RoleBinding` links the `pod-reader` Role to the `dev-user` ServiceAccount, so that identity now actually has those permissions — and only within the `dev` namespace, since RoleBindings (unlike ClusterRoleBindings) are namespace-scoped.

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

📸 **Screenshot 7:** The three `kubectl auth can-i` results (YES / NO / NO).

**Report answer — Authentication vs. Authorization:**
In all three checks, the service account is successfully **authenticated** — Kubernetes recognizes `system:serviceaccount:dev:dev-user` as a valid, known identity in every case; that step never fails. What differs is **authorization**: for `list pods -n dev`, the RBAC Role+RoleBinding explicitly grants that verb/resource/namespace combination, so it's authorized (YES). For `delete pods -n dev` and `list pods -n prod`, the identity is still authenticated, but the RBAC rules attached to it never grant a `delete` verb, nor any permissions in the `prod` namespace — so the **authorization** step is what blocks the request (NO/NO), not authentication. This demonstrates the RBAC principle of least privilege: identity is verified, but access is separately and precisely scoped.

---

## Verification Command

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```
**Why:** Dumps the full RoleBinding definition as YAML — proof that the binding (subject, role reference, namespace) exists exactly as configured, submitted as evidence RBAC is correctly in place.

---

## Deliverables Checklist

- [ ] Screenshot: `sts get-caller-identity` output
- [ ] Screenshot: `get-group Admins` showing `CloudAdmin_YOURNAME` as member
- [ ] Screenshot: `list-attached-user-policies` for Analyst (read-only policy only)
- [ ] Screenshot: Three `kubectl auth can-i` results (YES / NO / NO)
- [ ] Short-answer questions (Q1–Q5) answered in report
- [ ] `kubectl get rolebinding dev-user-binding -n dev -o yaml` output pasted

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

## Security Best-Practices Checklist

- [x] Root user is not used for daily tasks (a dedicated admin identity exists).
- [x] Permissions are granted via groups/roles, not directly to individual users.
- [x] At least one least-privilege (read-only) identity was created and tested.
- [x] Access keys were listed and a rotation (deactivate) was demonstrated.
- [x] Kubernetes RBAC blocks an unauthorised action (delete / cross-namespace).

---

## Cleanup & Teardown

```bash
# Remove the Kubernetes cluster
kind delete cluster --name ccse-lab1
```
**Why:** Tears down the entire local `kind` cluster and its Docker containers, freeing resources since the cluster was only needed for this lab exercise.

```bash
# Stop and remove LocalStack
docker stop localstack && docker rm localstack
```
**Why:** Stops the running LocalStack container and then removes the container itself, fully cleaning up the emulated AWS environment used in Session A.


---