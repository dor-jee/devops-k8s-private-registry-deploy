# Deploy an Application to Kubernetes from a Private Docker Registry (AWS ECR)

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Minikube](https://img.shields.io/badge/Minikube-326CE5?style=flat&logo=kubernetes&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS%20ECR-FF9900?style=flat&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat&logo=docker&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white)

A hands-on DevOps project that shows how to securely pull a container image from a **private AWS ECR repository** into a **Kubernetes cluster** (Minikube) using a `docker-registry` Secret and `imagePullSecrets`.

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




## Author

**N.Dorjee**
*CloudBuildOps — DevOps portfolio project*

[![GitHub](https://img.shields.io/badge/GitHub-dor--jee-181717?logo=github&logoColor=white)](https://github.com/dor-jee)
[![GitLab](https://img.shields.io/badge/GitLab-CloudBuildOps-FC6D26?logo=gitlab&logoColor=white)](https://gitlab.com/cloudbuildops/)
[![Website](https://img.shields.io/badge/Website-dorjee.dev-0A66C2?logo=googlechrome&logoColor=white)](https://dorjee.dev)
