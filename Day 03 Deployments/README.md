# ☸️ Kubernetes Masterclass: Deployments, DaemonSets & StatefulSets

[![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![DevOps](https://img.shields.io/badge/devops-red?style=for-the-badge&logo=azure-devops&logoColor=white)](https://en.wikipedia.org/wiki/DevOps)
![Kubernetes Deployment](https://img.shields.io/badge/Kubernetes_Deployment-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![ReplicaSet](https://img.shields.io/badge/Kubernetes_ReplicaSet-1F6FEB?style=for-the-badge&logo=kubernetes&logoColor=white)
![kubectl](https://img.shields.io/badge/kubectl-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Pods](https://img.shields.io/badge/Kubernetes_Pods-1F6FEB?style=for-the-badge&logo=kubernetes&logoColor=white)
![Nodes](https://img.shields.io/badge/Kubernetes_Nodes-0B3D91?style=for-the-badge&logo=kubernetes&logoColor=white)
![Control Plane](https://img.shields.io/badge/Kubernetes_Control_Plane-1D4ED8?style=for-the-badge&logo=kubernetes&logoColor=white)
![Worker Node](https://img.shields.io/badge/Kubernetes_Worker_Node-2563EB?style=for-the-badge&logo=kubernetes&logoColor=white)


Lets deep dive into Kubernetes orchestration. While Pods are the smallest unit of execution, **Deployments** are what 99.99% of production workloads actually run on. This repository is a hands-on guide to managing stateless and stateful applications at scale.

## 🚀 Why This Matters?
In a real-world production environment, we never manually manage individual pods. We need high availability, automated rollouts, and self-healing. This module demonstrates how Kubernetes handles these requirements using Deployments, ensure cluster-wide coverage with DaemonSets, and manage persistent identities with StatefulSets.

---

## 🛠️ Quick Start: The Imperative Way
Sometimes you just need a pod up and running for a quick network test or to check an image.

```bash
# Spin up a quick nginx pod for testing
kubectl run testpod1 --image nginx:latest
```
*Tip: Use this for debugging only. In production, always use Declarative YAML.*

---

## 🏗️ Mastering Deployments
A Deployment provides declarative updates for Pods and ReplicaSets. It's the "brain" that ensures your desired state (e.g., "I want 6 replicas") is always maintained.

### The Hierarchy
`Deployment` ➡️ `ReplicaSet` ➡️ `Pods`

### Creating a Deployment
You can start with an imperative command to generate a template or create it directly:

```bash
# Generate a deployment YAML with 3 replicas
kubectl create deployment testapp --image dummyrepo/kubegame:v1 --replicas 3 --dry-run=client -o yaml
```

### Declarative Configuration (The DevOps Standard)
We use a structured YAML approach to manage our applications. This allows for version control and consistent environments.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: testpod1
  labels:
    app: testpod1
  annotations:
    info: "Internal Deployment for Day-04"
    owner: "devops-team@example.local"
spec:
  minReadySeconds: 10
  replicas: 6
  selector:
    matchLabels:
      app: testpod1
  template:
    metadata:
      labels:
        app: testpod1
    spec:
      containers:
        - name: kubegame
          image: dummyrepo/kubegame:v2
        - name: mysqldb
          image: dummyrepo/mydb:v1
          env:
            - name: DB_NAME
              value: "mysqldb"
            - name: DB_PORT
              value: "3306"
            - name: DB_URL
              value: "db.example.local"
```

### Labels vs Annotations
*   **Labels**: Used for selecting pods and organizing resources. (e.g., `app: testpod1`).
*   **Annotations**: Used for non-identifying metadata. Great for documentation, owner info, or tool-specific config (e.g., `owner: devops-team`).

---

## 🔐 Environment Variables
Managing configuration via environment variables is a core principle of cloud-native apps (12-Factor App). 

**How to verify they are set:**
1. Describe the pod: `kubectl describe pod <podname>`
2. Shell into the pod:
   ```bash
   kubectl exec -it <podname> -- bash
   # Then run:
   env
   ```

---

## 🌐 Exposing Your Apps (Services)
A Deployment creates pods, but how do we reach them? We use **Services**.

```bash
# Expose port 8000 externally, mapping to port 80 on the container
kubectl expose deployment testapp --name sv1 --port 8000 --target-port 80 --type NodePort

# Expose another service for internal database access
kubectl expose deployment testapp --name sv2 --port 5000 --target-port 5000 --type NodePort
```

**What is NodePort?**
Think of it as opening a specific port (30000-32767) on *every* Node in your cluster. Traffic hitting that port on any node is forwarded to your service.

---

## 🔄 Deployment Strategy: Rolling Updates
Standard in production, a **Rolling Update** allows you to update your application (from v1 to v2) without downtime.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Max 1 extra pod during update
    maxUnavailable: 0%   # Ensure all pods are always available
```

*   **maxSurge**: How many extra pods can be created during the update.
*   **maxUnavailable**: How many pods can be down during the process.
*   **minReadySeconds**: How long to wait after a pod is ready before continuing the rollout.

**Live Edit:**
```bash
kubectl edit deployment testapp
```
*Note: Use `kubectl explain deployment.spec.strategy` for the full technical breakdown.*

---

## 📦 DaemonSets & StatefulSets

### 🛡️ DaemonSets
Ensures that all (or some) Nodes run a copy of a Pod.
*   **Use Case**: Log collectors (fluentd), Monitoring agents (Prometheus node-exporter).
*   **Logic**: 1 Pod per Node automatically.

## 📦 DaemonSet Manifest Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: testpod1
  labels:
    app: testpod1
spec:
  selector:
    matchLabels:
      app: testpod1
  template:
    metadata:
      labels:
        app: testpod1
    spec:
      containers:
        - name: kubegame
          image: rayeez/kubegame:v2
```
---
### 💾 StatefulSets
Used for applications that require a stable identity.
*   **Use Case**: Databases (MySQL, PostgreSQL, MongoDB).
*   **Why?**: Pods get persistent hostnames (`db-0`, `db-1`), Network Identities and Indexing that survive restarts, and they attach to specific Persistent Volumes.

---

## 🧐 The Interview Corner

**Q: What is the difference between Deployments, DaemonSets, and StatefulSets?**

| Feature | Deployment | DaemonSet | StatefulSet |
| :--- | :--- | :--- | :--- |
| **Logic** | Manages N replicas anywhere | 1 Pod on every Node | N replicas with fixed order/identity |
| **Scalability** | Horizontal (ReplicaCount) | Automatic per Node | Ordered (0, 1, 2...) |
| **Common Use** | Web Apps, APIs (Stateless) | Logs, Monitoring | Databases, Queues (Stateful) |
| **Network** | Random Pod Names | Random Pod Names | Stable Hostnames (index based) |

---

## 📖 Command Reference
A summary of the commands executed during this module:

```bash
kubectl get pods
kubectl get deploy
kubectl get pods -o wide
kubectl apply -f deployment.yaml
kubectl delete deployment testpod1
kubectl explain deployment.spec
kubectl edit deployment testpod1
kubectl delete daemonset testpod1
```

---
## ✅ Conclusion

This repository provides a practical and hands-on understanding of how Kubernetes workloads are managed in real-world environments using **Deployments**, **DaemonSets**, and **StatefulSets**.

By working through these examples, you now understand:

- How Deployments manage ReplicaSets and Pods for stateless applications  
- How rolling updates ensure zero-downtime deployments  
- How to configure strategies like `maxSurge`, `maxUnavailable`, and `minReadySeconds`  
- How to configure environment variables inside containers  
- Why DaemonSets are essential for node-level workloads like monitoring and logging  
- Why StatefulSets are critical for databases and stateful applications  

This repository is not just about running commands — it reflects how Kubernetes is used in real production environments.

If you're preparing for interviews, building real-world projects, or strengthening your DevOps foundation, mastering these workload types is essential.

Keep exploring. Keep building. Keep deploying. 🚀
