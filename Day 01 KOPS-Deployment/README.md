# ☸️ Kubernetes & KOPS: Production-Grade Hands-on Learning

[![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/)
[![KOPS](https://img.shields.io/badge/KOPS-%232496ED.svg?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kops.sigs.k8s.io/)
![EC2](https://img.shields.io/badge/EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white)
![S3 Bucket](https://img.shields.io/badge/S3_Bucket-569A31?style=for-the-badge&logo=amazons3&logoColor=white)
![Route53](https://img.shields.io/badge/Route_53-3F51B5?style=for-the-badge&logo=amazonroute53&logoColor=white)
![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Pods](https://img.shields.io/badge/Kubernetes_Pods-1F6FEB?style=for-the-badge&logo=kubernetes&logoColor=white)
![Nodes](https://img.shields.io/badge/Kubernetes_Nodes-0B3D91?style=for-the-badge&logo=kubernetes&logoColor=white)
![Control Plane](https://img.shields.io/badge/Kubernetes_Control_Plane-1D4ED8?style=for-the-badge&logo=kubernetes&logoColor=white)
![Worker Node](https://img.shields.io/badge/Kubernetes_Worker_Node-2563EB?style=for-the-badge&logo=kubernetes&logoColor=white)

A comprehensive, hands-on repository documenting the journey of mastering **Kubernetes** using **KOPS (Kubernetes Operations)**. This repository bridges the gap between basic container orchestration and production-ready cluster management on AWS.

---

## 🚀 Overview

This project demonstrates a real-world DevOps workflow for deploying and managing a Kubernetes cluster. It covers everything from setting up a management server to complex networking concepts like Pod IP isolation.

### Key Highlights
- **Infrastructure as Code**: Managing AWS clusters via KOPS manifests.
- **Cluster Lifecycle**: Dry-runs, updates, and zero-downtime scaling.
- **Deep Dive**: In-depth exploration of Namespaces, Pod types, and Control Plane internals.
- **Real-time Production Concepts**: Leveraging Auto Scaling Groups for high availability.

---

## 🛠️ Environment Setup & Prerequisites

Before diving into Kubernetes, we need to prepare our **Management Server** (Ubuntu 24.04 recommended).

### 1. Management Server Preparation
We use a dedicated EC2 instance (t3.medium) to manage the cluster. No direct login to the nodes is required—everything is handled from here.

- **IAM Role**: Attach an IAM role with `AdministratorAccess` to the Management Server.
- **SSH Keys**: Generate keys specifically for the cluster.
  ```bash
  ssh-keygen
  ```

### 2. Tool Installation
Install `kubectl` and `kops` to the system path:
```bash
# Install KOPS
wget https://github.com/kubernetes/kops/releases/download/v1.34.0/kops-linux-amd64
chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

### 3. Shell Configuration
Boost productivity by adding aliases and environment variables to your `~/.bashrc`:
```bash
# Environment Variables
export NAME=example-cluster.com
export KOPS_STATE_STORE=s3://my-k8s-state-bucket
export AWS_REGION=us-east-1
export CLUSTER_NAME=example-cluster.com

# Shortcuts
alias ku='kubectl'

# Apply changes
source ~/.bashrc
```

---

## 🏗️ KOPS Cluster Lifecycle

KOPS is the best tool for deploying production-grade clusters on AWS because it automatically configures **Auto Scaling Groups (ASG)**. If a node fails or is deleted, the ASG will instantly recreate it.

### Cluster Creation Steps
1. **Dry Run Manifest Generation**:
   ```bash
   kops create cluster --name=${NAME} \
     --state=${KOPS_STATE_STORE} --zones=us-east-1a,us-east-1b \
     --node-count=2 --control-plane-count=1 --node-size=t3.medium \
     --control-plane-size=t3.medium --ssh-public-key ~/.ssh/id_ed25519.pub \
     --dns-zone=${NAME} --dry-run --output yaml > cluster.yml
   ```
2. **Manifest Customization**:
   Modify `cluster.yml` to adjust subnet CIDRs for better isolation:
   - Subnet A: `172.20.1.0/24`
   - Subnet B: `172.20.2.0/24`

3. **Deploy & Validate**:
   ```bash
   kops create -f cluster.yml
   kops update cluster --name ${NAME} --yes --admin
   kops validate cluster --wait 10m
   ```

### Verification
A healthy cluster should report:
- `1 Master Node`
- `2 Worker Nodes`

---

## 📦 Kubernetes Hands-on Guide

### 1. Smoke Testing (Sanity Check)
```bash
ku get nodes         # List all nodes
ku cluster-info      # Check cluster status
ku get ns            # List namespaces
```

### 2. Namespace & Isolation
Namespaces allow multiple teams (e.g., Alpha, Bravo, Charlie) to work on the same cluster without interference.
- **Tip**: Communication between namespaces can be restricted or allowed using **Network Policies**.

### 3. Pod Concepts
A Pod is the smallest deployable unit. It can contain:
- **Single Container**: One app, one container.
- **Multi-Container**: Main app + **Sidecar** (logs) or **Proxy** (networking).

### 4. Imperative vs. Declarative
| Feature | Imperative | Declarative |
| :--- | :--- | :--- |
| **Command** | `ku run testpod1 --image=nginx` | `ku apply -f pod.yaml` |
| **Pros** | Quick, good for testing | Reproducible, version controlled |
| **Real-time Tip** | Use `--dry-run=client -o yaml` to generate manifests quickly. |

### 5️⃣ Realtime Inline Deployment Approach

In realtime, I sometimes use an inline YAML method directly from the terminal:

```bash
echo '
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: testpod3
  name: testpod3
spec:
  containers:
    - image: nginx:latest
      name: testpod3
' | ku apply -f -
```bash

---
```

---

## 🔍 Under the Hood: Control Plane

Inspect the brain of your cluster in the `kube-system` namespace:
```bash
ku get pods -n kube-system -o wide | grep -E 'api|etcd|scheduler|controller'
```
**Observation**: All control plane components run on the same private IP address of the Master Node, ensuring low latency and tight integration.

---

## 📁 Repository Structure
```text
.
├── cluster.yml         # KOPS cluster configuration
├── manifests/          # Kubernetes YAML declarations
│   └── testpod1.yml    # Sample Nginx Pod
├── scripts/            # Setup and utility scripts
└── README.md           # You are here!
```

---

## 👋 Conclusion

This repository serves as a blueprint for anyone transitioning from local K8s experimentation to cloud-native cluster management. By combining the power of **KOPS** with structured **kubectl** operations, we achieve a scalable and resilient infrastructure.

**Happy Architecting!** 🚀
