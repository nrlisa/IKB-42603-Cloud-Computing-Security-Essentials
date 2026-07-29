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
<div align="center">
<img width="612" height="55" alt="Screenshot 2026-07-29 115703" src="https://github.com/user-attachments/assets/e3f28b15-26f0-499d-9547-2cbaf4719f8e" />
<img width="653" height="402" alt="Screenshot 2026-07-29 115857" src="https://github.com/user-attachments/assets/478fdccd-4ad2-46e1-9534-92ef7f85596a" />
</div>

---

## 2. AWS CLI

The AWS Command Line Interface (v2) was installed using the official MSI installer (`AWSCLIV2-3.14.6.msi`) and verified:

```text
aws --version
aws-cli/2.36.8 Python/3.14.6 Linux/6.18.12+kali-amd64 exe/x86_64.kali.2026
```

* **Result:** AWS CLI v2 is installed and accessible via system path.
* **Evidence:**
<div align="center">
<img width="252" height="135" alt="Screenshot 2026-07-29 120305" src="https://github.com/user-attachments/assets/65386925-0e28-4d28-bab4-1a202d84abdd" />
<img width="615" height="55" alt="Screenshot 2026-07-29 120140" src="https://github.com/user-attachments/assets/32ca5473-685d-434c-82e7-54eac411003e" />
</div>

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
<div align="center">
<img width="252" height="135" alt="Screenshot 2026-07-29 120305" src="https://github.com/user-attachments/assets/14249332-6b29-4340-8725-aeb0fb4e1e93" />
</div>

---

## 4. Helper Tools

Git Bash was installed from the official repository to provide a Unix-like terminal environment for executing lab scripts and commands.

* **Result:** Helper terminal environment is fully operational.
* **Evidence:**
<div align="center">
<img width="665" height="191" alt="Screenshot 2026-07-29 120411" src="https://github.com/user-attachments/assets/68c53674-f8db-4b0b-8d40-16403998e813" />
</div>

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
<div align="center">
<img width="612" height="55" alt="Screenshot 2026-07-29 115703" src="https://github.com/user-attachments/assets/5c16c78e-62c1-46ec-898e-72716326c49c" />
<img width="786" height="388" alt="Screenshot 2026-07-27 195057" src="https://github.com/user-attachments/assets/0dc15af4-9538-43bf-924c-cff73a142a3f" />
<img width="577" height="95" alt="Screenshot 2026-07-27 194732" src="https://github.com/user-attachments/assets/0b74d738-ef7f-414a-baee-d7f5029f4c7f" />
</div>



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
<div align="center">
<img width="768" height="172" alt="Screenshot 2026-07-27 201151" src="https://github.com/user-attachments/assets/7e5b7c69-c07c-402d-bc8a-18692f78ff42" />

<img width="302" height="63" alt="Screenshot 2026-07-27 201208" src="https://github.com/user-attachments/assets/f767c6ff-81f0-4062-81f5-71bfbdc2e5c1" />
</div>

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
<div align="center">
<img width="427" height="331" alt="Screenshot 2026-07-27 201321" src="https://github.com/user-attachments/assets/29649ec1-5220-43d8-b324-a81b367659b2" />
</div>

---

## Conclusion

The Lab 0 environment setup is complete and fully verified. All required tooling—including Docker, AWS CLI, `kind`, `kubectl`, Git Bash, LocalStack, and target endpoints—is configured and functioning as expected.
