# Lab 1 — Commands Explained (Easy Words)

A plain-words guide to every command in **Lab 1** (`Lab 1.md`). It's split into the same sections as the report.

## Big Picture

Lab 1 does two things:

- **Session A** — uses a fake AWS (LocalStack) to practice identity management: users, groups, policies, access keys.
- **Session B** — uses a fake Kubernetes cluster (kind) to practice access control with RBAC.

---

## Session A — Setup (getting the fake AWS running)

| Command | What it does, in easy words |
|---|---|
| `docker --version` | Checks that Docker is installed before we try anything. |
| `docker run -d --name localstack -p 4566:4566 localstack/localstack` | Starts the fake-AWS container. `-d` = run in background; `--name localstack` = give it a nickname; `-p 4566:4566` = let your computer talk to it on port 4566. |
| `curl http://localhost:4566/_localstack/health` | Asks LocalStack "are you awake?" — it lists which fake AWS services are running. |
| `aws configure set aws_access_key_id test` (×3) | Stores dummy login info (`test`/`test`, region `us-east-1`). The AWS command-line tool refuses to work without *some* credentials, and LocalStack accepts anything. |
| `aws --endpoint-url=http://localhost:4566 sts get-caller-identity` | "Who am I right now?" — confirms the AWS tool is talking to LocalStack (account `000000000000`), not real AWS. |
| `EP='--endpoint-url=http://localhost:4566'` | Creates a shortcut: `EP` stands for "talk to LocalStack". Every later command just types `$EP` instead of the long URL. |

---

## Task 2 — Create an admin that isn't "root"

| Command | What it does |
|---|---|
| `aws $EP iam create-group --group-name Admins` | Makes an empty team called `Admins`. |
| `aws $EP iam attach-group-policy ... --policy-arn ...AdministratorAccess` | Gives the *whole team* the "do everything" permission (AdministratorAccess) instead of giving it to one person. |
| `aws $EP iam list-attached-group-policies --group-name Admins` | Asks "what permissions does the Admins team have?" — proof the policy is attached. |
| `aws $EP iam create-user --user-name CloudAdmin_Lisa` | Creates a personal login (`CloudAdmin_Lisa`) so you stop using the all-powerful root account. |
| `aws $EP iam add-user-to-group --group-name Admins --user-name CloudAdmin_Lisa` | Puts Lisa **into** the Admins team. Now she inherits the team's powers automatically. |
| `aws $EP iam get-group --group-name Admins` | "Who's in the Admins team?" — proof Lisa is a member. |

**Why groups:** you manage permissions in one place (the team), and members come and go by joining/leaving.

---

## Task 3 — A read-only teammate

| Command | What it does |
|---|---|
| `aws $EP iam create-user --user-name Analyst_Raisha` | Creates a second login for a teammate. |
| `aws $EP iam attach-user-policy --user-name Analyst_Raisha --policy-arn ...AmazonS3ReadOnlyAccess` | Gives Raisha *only* "look but don't touch" permission on S3 files (least privilege). |
| `aws $EP iam list-attached-user-policies --user-name Analyst_Raisha` | Shows the single read-only policy attached to her — proof she can't delete or write. |

---

## Task 4 — Access keys (passwords for programs)

| Command | What it does |
|---|---|
| `aws $EP iam create-access-key --user-name Analyst_Raisha` | Generates a secret "program key" (ID + secret) so a script can log in as Raisha without a console password. |
| `aws $EP iam list-access-keys --user-name Analyst_Raisha` | Lists her keys and their status (Active/Inactive). |
| `aws $EP iam update-access-key --user-name Analyst_Raisha --access-key-id <ID> --status Inactive` | Switches one key off — that's "rotation": if a key leaks, you turn it off and make a new one. |

---

## Session B — Kubernetes access control

| Command | What it does |
|---|---|
| `kind create cluster --name ccse-lab1` | Creates a small, disposable Kubernetes cluster in Docker (free practice cluster). |
| `kubectl cluster-info --context kind-ccse-lab1` | "Is the cluster's brain reachable?" |
| `kubectl get nodes` | Lists the cluster's computers (nodes) — should show `Ready`. |
| `kubectl create namespace dev` / `prod` | Makes two separate rooms (`dev`, `prod`) inside the cluster. |
| `kubectl get namespaces` | Lists all rooms — proof both exist. |
| `kubectl create serviceaccount devsofia -n dev` | Creates a "robot identity" called `devsofia` that represents a developer, inside the `dev` room. |
| `kubectl create role pod-reader -n dev --verb=get,list,watch --resource=pods` | Writes a job description (`role`) saying: in the `dev` room you may only *look at* pods (get/list/watch) — nothing else. |
| `kubectl create rolebinding dev-user-binding -n dev --role=pod-reader --serviceaccount=dev:devsofia` | Hires `devsofia` for that job: "this identity has the pod-reader job in the dev room". A role does nothing until it's *bound* to someone. |
| `kubectl auth can-i list pods -n dev --as=$SA` | "Can this identity list pods?" — answers `yes`/`no` without actually doing it. The `--as=` says "pretend you're this identity". |
| `kubectl auth can-i delete pods -n dev --as=$SA` | Same check but for deleting — answers `no` (not in the job description). |
| `kubectl auth can-i list pods -n prod --as=$SA` | Same check in the other room — answers `no`, because the job is only in `dev`. |
| `kubectl get rolebinding dev-user-binding -n dev -o yaml` | Prints the hiring contract as YAML — evidence that the binding is exactly as configured. |

---

## Cleanup

| Command | What it does |
|---|---|
| `kind delete cluster --name ccse-lab1` | Destroys the practice cluster. |
| `docker stop localstack && docker rm localstack` | Stops and removes the fake AWS container. |

---

## One-line takeaway

Session A = who can log in and what they may do (IAM users/groups/policies/keys); Session B = the same idea inside Kubernetes, where a Role says *what*, a RoleBinding says *who gets it*, and `kubectl auth can-i` is the test that proves the walls actually hold.
