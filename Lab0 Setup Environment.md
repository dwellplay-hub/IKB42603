# Lab 0: Environment Setup Instructions

## Course Information

- Course: IKB42603 Cloud Computing Security Essentials
- Lab: Lab 0 - Environment Setup
- Student Name: Muhammad Danish Isyraq Bin Ab Ghani
- Date: 29 July 2026

## Objective

The goal of this lab is to prepare a local environment that can support Docker, AWS CLI v2, kind, kubectl, OpenSSL, oathtool, LocalStack, and a local Kubernetes cluster. The setup follows the guide in IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf and uses LocalStack as a local AWS-compatible endpoint.

## Step-by-step Procedure

### Step 1: Install and verify Docker

Docker is required to run container-based tools such as LocalStack and the kind Kubernetes cluster.

Follow these instructions:

1. Open a terminal on your machine.
2. Check that Docker is installed by running:

```bash
docker --version
```

3. Verify that Docker can run a container successfully by executing:

```bash
docker run --rm hello-world
```

4. Confirm that the command prints a successful version and that the hello-world container runs without errors.
5. Save the output as proof of completion.

Expected evidence:

- The Docker version is displayed.
- The hello-world container runs successfully.

Proof of completion:

<img width="668" height="372" alt="docker --version prints a version, and docker run hello-world works" src="https://github.com/user-attachments/assets/92ca7eff-cae1-4bc5-a96b-95f48aa616c8" />


### Step 2: Install and verify AWS CLI v2

AWS CLI v2 is used to interact with LocalStack during the labs.

Follow these instructions:

1. Open a terminal.
2. Check the AWS CLI version using:

```bash
aws --version
```

3. Confirm that the output shows AWS CLI version 2.x.
4. Keep the output as evidence.

Expected evidence:

- AWS CLI v2 is installed successfully.

Proof of completion:

<img width="685" height="137" alt="aws --version prints aws-cli2 x" src="https://github.com/user-attachments/assets/73fe1216-4869-4beb-bfdb-948ce67a347f" />

### Step 3: Install and verify kind and kubectl

kind creates a local Kubernetes cluster inside Docker, while kubectl is used to manage it.

Follow these instructions:

1. Open a terminal.
2. Check the version of kind:

```bash
kind --version
```

3. Check the kubectl client version:

```bash
kubectl version --client
```

4. Confirm that both commands return version information without errors.

Expected evidence:

- kind is installed and working.
- kubectl is installed and working.

Proof of completion:

<img width="287" height="200" alt="kind --version and kubectl version --client both work" src="https://github.com/user-attachments/assets/d45e998b-7b9f-41a1-941a-89015ab26200" />

### Step 4: Install and verify helper tools

OpenSSL and oathtool are required for later labs.

Follow these instructions:

1. Open a terminal.
2. Verify OpenSSL:

```bash
openssl version
```

3. Verify oathtool:

```bash
oathtool --version
```

4. Confirm that both commands return valid version information.

Expected evidence:

- OpenSSL is installed.
- oathtool is installed.

### Step 5: Start and verify LocalStack

LocalStack simulates AWS services locally so the labs can run without a real AWS account.

Follow these instructions:

1. Start LocalStack in a container:

```bash
docker run -d --name localstack -p 4566:4566 localstack/localstack:3.0
```

2. Check that the service is healthy:

```bash
curl http://localhost:4566/_localstack/health
```

3. Verify that the container is running:

```bash
docker ps
```

4. Confirm that LocalStack responds successfully and the container is listed.

Expected evidence:

- The LocalStack health endpoint responds successfully.

Proof of completion:

<img width="675" height="422" alt="LocalStack starts and curl  health responds" src="https://github.com/user-attachments/assets/11bddc8e-2709-403a-953f-5dbf91412cd2" />

### Step 6: Create and verify the Kubernetes cluster

A local kind cluster named ccse should be created and verified.

Follow these instructions:

1. Create the cluster using:

```bash
kind create cluster --name ccse
```

2. Verify that the cluster nodes are available:

```bash
kubectl get nodes
```

3. Confirm that the output shows a node in the Ready state.

Expected evidence:

- A local Kubernetes cluster has been created successfully.

Proof of completion:

<img width="505" height="347" alt="kind create cluster works and kubectl get nodes shows a node" src="https://github.com/user-attachments/assets/97c537c8-00e5-4c62-be22-d1ef981e4954" />

### Step 7: Configure AWS CLI for LocalStack

The AWS CLI should be configured with dummy credentials and pointed to the LocalStack endpoint.

Follow these instructions:

1. Set dummy AWS credentials:

```bash
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1
```

2. Define the LocalStack endpoint:

```bash
EP='--endpoint-url=http://localhost:4566'
```

3. Test the configuration by calling STS through LocalStack:

```bash
aws $EP sts get-caller-identity
```

4. Confirm that the command returns an identity or account information successfully.

Expected evidence:

- AWS CLI can reach LocalStack successfully.

Proof of completion:

<img width="431" height="305" alt="aws $EP sts get-caller-identity returns an identity" src="https://github.com/user-attachments/assets/f68262c6-4ad6-4ac3-be30-b6fa5479f4b2" />

## Verification Checklist

- Docker works and `docker run hello-world` succeeds.
- AWS CLI v2 is installed and responds correctly.
- kind and kubectl are installed and report versions.
- LocalStack health endpoint returns successfully.
- A kind Kubernetes cluster is running and exposes a ready node.
- AWS CLI can call STS through LocalStack and return an identity.

## Conclusion

The environment setup completed successfully. The local lab environment is ready for the next lab activities, including container-based services, LocalStack, and a local Kubernetes cluster.
