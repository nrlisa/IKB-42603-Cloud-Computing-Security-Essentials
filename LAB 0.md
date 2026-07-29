# Lab 0: Environment Setup Report

**Name:** Nurlisa Sofiya Binti Mahadzir

**Subject:** Cloud Computing Security Essentials

**Code:** IKB 42603

**Date:** 28 July 2026

---

## Purpose

This report documents the local development environment setup and verification for Windows 11. The accompanying screenshots serve as evidence for each completed check.

## System Requirements

* **RAM:** Minimum 16 GB
* **Storage:** At least 10 GB free space

---

## 1. Docker

Docker Desktop was installed from the official website (ensuring WSL2 backend integration was selected during setup) and verified via the command line:

```text
docker --version
Docker version 29.6.2, build dfc4efb

```

A test container was executed to confirm image pulling and container runtime functionality:

```text
docker run --rm hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.

```

* **Result:** Docker Engine is operational and successfully pulling and running containers.
* **Evidence:**

---

## 2. AWS CLI

The AWS Command Line Interface (v2) was installed using the official MSI installer (`AWSCLIV2-3.14.6.msi`) and verified:

```text
aws --version
aws-cli/2.36.9 Python/3.14.6 Windows/11 exe/AMD64

```

* **Result:** AWS CLI v2 is installed and accessible via system path.
* **Evidence:**

---

## 3. kind and kubectl

Kubernetes-in-Docker (`kind`) and the Kubernetes command-line client (`kubectl`) were installed via Chocolatey package manager (`choco`) and verified:

```text
kind --version
kind version 0.31.0

kubectl version --client
Client Version: v1.36.3
Kustomize Version: v5.8.1

```

* **Result:** Both `kind` and `kubectl` are configured and ready for local cluster operations.
* **Evidence:**

---

## 4. Helper Tools

Git Bash was installed from the official repository to provide a Unix-like terminal environment for executing lab scripts and commands.

* **Result:** Helper terminal environment is fully operational.
* **Evidence:**

---

## 5. Start and Stop Lab Environment

### Local AWS (LocalStack)

LocalStack was initialized via Docker on port `4566` to emulate AWS services locally without requiring cloud credentials:

```text
docker run -d --name localstack -p 4566:4566 localstack/localstack:4.14.0

```

Health endpoint verification:

```text
curl http://localhost:4566/_localstack/health

```

Container lifecycle operations were tested:

```text
docker stop localstack
docker start localstack
docker rm -f localstack

```

* **Result:** LocalStack container runs reliably, passes health checks, and is visible within Docker Desktop.
* **Evidence:**

### Local Kubernetes Cluster

A test cluster named `ccse` was instantiated using `kind`:

```text
kind create cluster --name ccse

```

Cluster status verification:

```text
kubectl cluster-info --context kind-ccse
kubectl get nodes

```

The output confirmed the Kubernetes control plane and CoreDNS services were running, with node `ccse-control-plane` reporting `Ready` (version `v1.35.0`).

Cluster cleanup test:

```text
kind delete cluster --name ccse

```

* **Result:** Kubernetes local cluster creation, node inspection, and tear-down verified successfully.
* **Evidence:**

---

## 6. AWS CLI Configuration for LocalStack

The AWS CLI was configured within Git Bash using mock credentials to target the local environment:

```text
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

```

Endpoint alias set for LocalStack targeting:

```text
EP='--endpoint-url=http://localhost:4566'

```

Identity verification via AWS STS:

```text
aws $EP sts get-caller-identity

```

Output confirmed local test identity (`Account: 000000000000`, `Arn: arn:aws:iam::000000000000:root`).

* **Result:** AWS CLI is successfully routed to interact with LocalStack.
* **Evidence:**

---

## Conclusion

The Lab 0 environment setup is complete and fully verified. All required tooling—including Docker, AWS CLI, `kind`, `kubectl`, Git Bash, LocalStack, and target endpoints—is configured and functioning as expected.