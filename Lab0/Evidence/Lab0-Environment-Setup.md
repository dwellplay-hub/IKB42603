# Lab 0: Environment Setup Report

## Course Information

- Course: IKB42603 Cloud Computing Security Essentials
- Lab: Lab 0 - Environment Setup
- Student Name: Student name
- Date: 29 July 2026

## Objective

The goal of this lab is to prepare a local environment that can support Docker, AWS CLI v2, kind, kubectl, OpenSSL, oathtool, LocalStack, and a local Kubernetes cluster. The setup follows the guide in IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf and uses LocalStack as a local AWS-compatible endpoint.

## Step-by-step Procedure

### Step 1: Install and verify Docker

Docker is required to run container-based tools such as LocalStack and the kind Kubernetes cluster.

Commands used:

```bash
docker --version
docker run --rm hello-world
```

Evidence:

<img src="docker --version prints a version, and docker run hello-world works..png" alt="Docker verification" width="700" />

### Step 2: Install and verify AWS CLI v2

AWS CLI v2 is used to interact with LocalStack during the labs.

Commands used:

```bash
aws --version
```

Evidence:

<img src="aws --version prints aws-cli2.x.png" alt="AWS CLI verification" width="700" />

### Step 3: Install and verify kind and kubectl

kind creates a local Kubernetes cluster inside Docker, while kubectl is used to manage it.

Commands used:

```bash
kind --version
kubectl version --client
```

Evidence:

<img src="kind --version and kubectl version --client both work..png" alt="kind and kubectl verification" width="700" />

### Step 4: Install and verify helper tools

OpenSSL and oathtool are required for later labs.

Commands used:

```bash
openssl version
oathtool --version
```

### Step 5: Start and verify LocalStack

LocalStack simulates AWS services locally so the labs can run without a real AWS account.

Commands used:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3.0
curl http://localhost:4566/_localstack/health
docker ps
```

Evidence:

<img src="LocalStack starts and curl ...health responds.png" alt="LocalStack health verification" width="800" />

### Step 6: Create and verify the Kubernetes cluster

A local kind cluster named ccse was created and verified.

Commands used:

```bash
kind create cluster --name ccse
kubectl get nodes
```

Evidence:

<img src="kind create cluster works and kubectl get nodes shows a node..png" alt="kind cluster verification" width="800" />

### Step 7: Configure AWS CLI for LocalStack

The AWS CLI was configured with dummy credentials and pointed to the LocalStack endpoint.

Commands used:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

Evidence:

<img src="aws $EP sts get-caller-identity returns an identity..png" alt="AWS CLI against LocalStack" width="800" />

## Verification Checklist

- Docker works and `docker run hello-world` succeeds.
- AWS CLI v2 is installed and responds correctly.
- kind and kubectl are installed and report versions.
- LocalStack health endpoint returns successfully.
- A kind Kubernetes cluster is running and exposes a ready node.
- AWS CLI can call STS through LocalStack and return an identity.

## Conclusion

The environment setup completed successfully. The local lab environment is ready for the next lab activities, including container-based services, LocalStack, and a local Kubernetes cluster.
