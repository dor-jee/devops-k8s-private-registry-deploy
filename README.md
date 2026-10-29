# Deploy an Application to Kubernetes from a Private Docker Registry (AWS ECR)

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Minikube](https://img.shields.io/badge/Minikube-326CE5?style=flat&logo=kubernetes&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS%20ECR-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white)

A hands-on DevOps project that shows how to securely pull a container image from a **private AWS ECR repository** into a **Kubernetes cluster** (Minikube) using a `docker-registry` Secret and `imagePullSecrets`.

---

## Table of Contents

- [Overview](#overview)
- [Architecture & Workflow](#architecture--workflow)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Step-by-Step Guide](#step-by-step-guide)
  - [1. Build and push the image to AWS ECR](#1-build-and-push-the-image-to-aws-ecr)
  - [2. Authenticate Docker against ECR](#2-authenticate-docker-against-ecr)
  - [3. Get the credentials into Minikube](#3-get-the-credentials-into-minikube)
  - [4. Create the registry Secret](#4-create-the-registry-secret)
  - [5. Create the Deployment](#5-create-the-deployment)
  - [6. Verify](#6-verify)
- [Troubleshooting](#troubleshooting)
- [Key Takeaways](#key-takeaways)
- [Author](#author)

---

## Overview

In a typical CI/CD workflow, the application is packaged and its image is pushed to a registry (AWS ECR, Nexus, Docker Hub, ...):

```
Commit code to Git  →  Trigger CI build (Jenkins)  →  Package application  →  Push image to registry
```

The open question is the last mile: **how does the Kubernetes cluster get permission to pull that image from a private registry?**

Private registries require explicit authentication. Kubernetes solves this in two steps:

1. Create a **Secret** that holds the registry credentials.
2. Reference that Secret in the Deployment/Pod via **`imagePullSecrets`**.

This project walks through both steps end to end using AWS ECR and Minikube.

## Architecture & Workflow

```
┌────────────┐   push   ┌───────────────────┐
│ CI / build │ ───────▶ │  AWS ECR (private)│
└────────────┘          └─────────▲─────────┘
                                  │ pull (authenticated)
                        ┌─────────┴─────────┐
                        │ Kubernetes cluster│
                        │  ┌─────────────┐  │
                        │  │   Secret    │  │  kubernetes.io/dockerconfigjson
                        │  └──────┬──────┘  │
                        │  ┌──────▼──────┐  │
                        │  │ Deployment  │  │  imagePullSecrets → Secret
                        │  └─────────────┘  │
                        └───────────────────┘
```

## Prerequisites

- An **AWS account** with a private **ECR repository** already created
- **AWS CLI** configured (`aws configure`)
- **Docker**
- **Minikube** and **kubectl**
- The sample Node.js app and its ECR build/push instructions: [aws-ecr-docker-registry](https://github.com/dor-jee/aws-ecr-docker-registry)

**Screenshot:** AWS ECR repository with the pushed image

<img src="screenshots/aws-ecr-image.png" alt="aws-ecr-image" width="600">

## Project Structure

```
.
├── README.md
├── my-app-deployment.yaml     # Deployment using imagePullSecrets
├── docker-secret.yaml         # Secret manifest (option 1)
└── docs/
    └── images/                # Screenshots
```

## Step-by-Step Guide

> Replace placeholders such as `<AWS_ACCOUNT_ID>`, `<REGION>` and `<IMAGE_NAME>` with your own values.

### 1. Build and push the image to AWS ECR

Follow the instructions in the [sample app repository](https://github.com/dor-jee/aws-ecr-docker-registry). Afterwards the image is available at:

```
<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<IMAGE_NAME>:<TAG>
```

### 2. Authenticate Docker against ECR

```bash
aws ecr get-login-password --region <REGION> | \
  docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
```

A successful login writes the credentials to `~/.docker/config.json`:

```bash
cat ~/.docker/config.json
```

**Screenshot:** AWS ECR login and `config.json`

<img src="screenshots/ecr-login-docker-config.png" alt="aecr-login-docker-config" width="600">

### 3. Get the credentials into Minikube

Minikube runs inside a VM/container, so it **cannot read the credentials stored on your local machine** (for example, macOS keeps them in the keychain). Instead, we log in to the registry from *inside* the Minikube node.

Get an auth token:

```bash
aws ecr get-login-password --region <REGION>
```

SSH into the Minikube node and confirm there is no `.docker` directory yet:

```bash
minikube ssh
ls -a
```

Log in from inside the node, pasting the token as the password:

```bash
docker login --username AWS -p <TOKEN> <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com
ls -a                          # a .docker directory now exists
cat .docker/config.json
```

Exit the node and copy `config.json` to your local machine, where `kubectl` runs:

```bash
exit
minikube cp minikube:/home/docker/.docker/config.json ~/.docker/config.json
```

### 4. Create the registry Secret

The Secret must be of type `kubernetes.io/dockerconfigjson` and **must live in the same namespace as the Deployment**.

#### Option 1: Declarative (manifest)

Base64-encode the config file:

```bash
cat ~/.docker/config.json | base64          # macOS
cat ~/.docker/config.json | base64 -w 0     # Linux (single line)
```

`docker-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-registry-key
data:
  .dockerconfigjson: <BASE64_ENCODED_CONFIG_JSON>
type: kubernetes.io/dockerconfigjson
```

```bash
kubectl apply -f docker-secret.yaml
```

#### Option 2: Imperative, from the config file

Creates the Secret directly in the cluster, without a manifest:

```bash
kubectl create secret generic my-registry-key \
  --from-file=.dockerconfigjson=$HOME/.docker/config.json \
  --type=kubernetes.io/dockerconfigjson
```

#### Option 3: Imperative, `docker-registry` shortcut

```bash
kubectl create secret docker-registry my-registry-key \
  --docker-server=<AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$(aws ecr get-login-password --region <REGION>)"
```

> **Which option should I use?** Options 1 and 2 reuse your `config.json`, which can hold credentials for **multiple registries** at once. Option 3 is fine when you only use a single registry.

Check that the Secret exists:

```bash
kubectl get secret
```

> **Screenshot:** `kubectl get secret`
>
<img src="screenshots/kubectl-create-secret-generic.png" alt="kubectl-create-secret-generic" width="600">

### 5. Create the Deployment

`my-app-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:
        - name: my-registry-key        # must match the Secret name
      containers:
        - name: my-app
          image: <AWS_ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com/<IMAGE_NAME>:<TAG>
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
```

Key points:

- **`imagePullSecrets`** tells Kubernetes which credentials to use when pulling the image.
- **`imagePullPolicy: Always`** forces a fresh pull on every Pod start, even if the image is already cached on the node.
- The **Secret name** in `imagePullSecrets` must match the Secret you created, and both must be in the **same namespace**.

Apply it:

```bash
kubectl apply -f my-app-deployment.yaml
```

### 6. Verify

```bash
kubectl get pods
kubectl describe pod <POD_NAME>     # check the Events section for the image pull
```

The Pod should reach the `Running` state.

> **Screenshot:** `kubectl get pods` showing the running application

<img src="screenshots/running-containers.png" alt="running-container" width="600">


## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `ErrImagePull` / `ImagePullBackOff` | Wrong image URL, tag, or Secret name | Check `kubectl describe pod` events and compare against the ECR URI |
| `no basic auth credentials` | Secret missing, in the wrong namespace, or not referenced | Confirm `kubectl get secret -n <namespace>` and `imagePullSecrets` |
| Pulls worked yesterday, fail today | **ECR tokens expire after 12 hours** | Re-run the login and recreate/update the Secret |
| Pod can't pull after editing the Secret | Existing Pods keep old credentials | `kubectl rollout restart deployment/my-app` |

## Key Takeaways

- Private registries need explicit credentials in Kubernetes: a `dockerconfigjson` Secret plus `imagePullSecrets`.
- A Secret is namespaced and only usable by Pods in the same namespace.
- ECR authorization tokens are short-lived (12 hours). For production, automate token refresh (for example with a CronJob), or use IAM roles for service accounts when running on EKS, instead of static credentials.
- Secrets are only base64-encoded, not encrypted. Never commit real credentials to Git.

## Author

**N.Dorjee**
*CloudBuildOps — DevOps portfolio project*

[![GitHub](https://img.shields.io/badge/GitHub-dor--jee-181717?logo=github&logoColor=white)](https://github.com/dor-jee)
[![GitLab](https://img.shields.io/badge/GitLab-CloudBuildOps-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com/cloudbuildops/)
[![Website](https://img.shields.io/badge/Website-dorjee.dev-0A66C2?logo=googlechrome&logoColor=white)](https://dorjee.dev)
