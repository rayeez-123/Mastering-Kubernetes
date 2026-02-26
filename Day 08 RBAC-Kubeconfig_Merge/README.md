# Kubernetes RBAC & Multi-User Access Control

## 🛠 Tech Stack

| Tool | Purpose |
|------|---------|
| **Kubernetes (kops)** | Cluster orchestration |
| **OpenSSL** | Certificate & key generation |
| **kubectl** | Kubernetes CLI |
| **kubectx** | Quick context switching |
| **Portainer** | Web-based Kubernetes UI |
| **RBAC** (`rbac.authorization.k8s.io/v1`) | Role-based access control |

---

## 📌 Concepts (Quick Reference)

- **Namespace** — Logical isolation within a cluster (e.g., `development`, `production`)
- **Role** — Grants permissions to resources *within a namespace*
- **ClusterRole** — Grants permissions *cluster-wide* across all namespaces
- **RoleBinding** — Binds a Role to a User within a namespace
- **ClusterRoleBinding** — Binds a ClusterRole to a User cluster-wide
- **kubeconfig** — Config file holding cluster, user, and context info for kubectl
- **Context** — A named combination of cluster + user + namespace in kubeconfig

---

## 🚀 Step-by-Step Setup

### Step 1 — SSH Key Setup
```bash
cd ~/.ssh/
ls -al    # Verify ed12259 pub/pvt keys exist
```

---

### Step 2 — Create Namespaces
```bash
kubectl create namespace development
kubectl create namespace production
```

---

### Step 3 — Get CA Certificates from Control Plane
```bash
# On control plane node
find / -name kops-controller    # → /etc/kubernetes/kops-controller
# Copy kubernetes-ca.crt and kubernetes-ca.key contents to management server
# Paste as /tmp/ca.crt and /tmp/ca.key on management server
```

---

### Step 4 — Generate User Certificates

#### user1 (development namespace)
```bash
openssl genrsa -out user1.key 2048
openssl req -new -key user1.key -out user1.csr -subj "/CN=user1/O=development"
openssl x509 -req -in user1.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user1.crt -days 365
```

#### user2 (production namespace)
```bash
openssl genrsa -out user2.key 2048
openssl req -new -key user2.key -out user2.csr -subj "/CN=user2/O=production"
openssl x509 -req -in user2.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user2.crt -days 365
```
> **Output:** `user1.crt`, `user1.key`, `user2.crt`, `user2.key`

---

### Step 5 — Copy Certs to Control Plane
```bash
# Copy user1.crt, user1.key, user2.crt, user2.key to /root/ on control plane
```

---

### Step 6 — Create kubeconfig Files

Copy `~/.kube/config` from management server as a base. Set client cert/key paths:
```
/root/user1.crt  →  client-certificate
/root/user1.key  →  client-key
```

Create and test configs on control plane:
```bash
export KUBECONFIG=/root/USER1-CONFIG
kubectl get pods    # → Forbidden (expected, no roles yet)

export KUBECONFIG=/root/USER2-CONFIG
kubectl get pods    # → Forbidden (expected, no roles yet)
```

---

### Step 7 — Apply RBAC from Management Server

#### Roles (namespace-scoped)
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user1-role
  namespace: development
rules:
  - apiGroups: ["", "apps", "networking.k8s.io"]
    resources: ["pods", "deployments", "replicasets", "nodes", "ingress", "services"]
    verbs: ["get", "update", "list", "create", "delete"]

---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user2-role
  namespace: production
rules:
  - apiGroups: ["", "apps", "networking.k8s.io"]
    resources: ["pods", "deployments", "replicasets", "nodes", "ingress", "services"]
    verbs: ["get", "update", "list", "create", "delete"]
```

#### RoleBindings
```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user1-RoleBinding
  namespace: development
subjects:
  - kind: User
    name: user1
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: user1-role
  apiGroup: rbac.authorization.k8s.io

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: user2-RoleBinding
  namespace: production
subjects:
  - kind: User
    name: user2
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: user2-role
  apiGroup: rbac.authorization.k8s.io
```

---

### Step 8 — Verify User Access

#### user1 → development only
```bash
export KUBECONFIG=/root/USER1-CONFIG

kubectl get pods                          # No resources in development namespace
kubectl create deployment development-pods --image rayeez/kubegame:v2
kubectl get pods                          # ✅ 1 pod running
kubectl get pods -n production            # ❌ Forbidden
```

#### user2 → production only
```bash
export KUBECONFIG=/root/USER2-CONFIG

kubectl get pods                                                        # No resources in production
kubectl create deployment production-pods --image nginx:latest --replicas 10
kubectl get pods                                                        # ✅ 10 pods running
kubectl get pods -n development                                         # ❌ Forbidden
```

---

### Step 9 — Admin User (rayeez) — Cluster-wide Access

#### Generate certs
```bash
openssl genrsa -out rayeez.key 2048
openssl req -new -key rayeez.key -out rayeez.csr -subj "/CN=rayeez/O=clusteradmin"
openssl x509 -req -in rayeez.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out rayeez.crt -days 365
```
> Copy `rayeez.crt`, `rayeez.key` to `/root/` on control plane, create `RAYEEZ-CONFIG`

#### ClusterRole & ClusterRoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: new-cluster-admin-role
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ClusterRole-rayeez
subjects:
  - kind: User
    name: rayeez
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: new-cluster-admin-role
  apiGroup: rbac.authorization.k8s.io
```

#### Verify admin access
```bash
export KUBECONFIG=/root/RAYEEZ-CONFIG

kubectl get pods                    # No resources in default namespace
kubectl get pods -n development     # ✅ 1 pod running
kubectl get pods -n production      # ✅ 10 pods running
```

---

### Step 10 — Merge Configs & Switch Contexts

```bash
# Merge all configs into one file
KUBECONFIG=USER1-CONFIG:USER2-CONFIG:RAYEEZ-CONFIG kubectl config view --merge --flatten > mixed-config.txt

# Install kubectx for easy context switching
# https://github.com/ahmetb/kubectx

# Switch contexts
kubectx rayeez-context
kubectx user1-context
kubectx user2-context
```

---

### Step 11 — Install Portainer (Web UI)

```bash
# Apply Portainer manifest from management server
kubectl apply -f https://raw.githubusercontent.com/portainer/k8s/master/deploy/manifests/portainer/portainer.yaml

# Open port 30779 on control plane node
# Access in browser:
https://<public-dns>:30779
```

---

## 🔐 Access Summary

| User | Namespace | Access |
|------|-----------|--------|
| `user1` | `development` | Full (pods, deployments, services, etc.) |
| `user2` | `production` | Full (pods, deployments, services, etc.) |
| `rayeez` | All namespaces | Cluster Admin (full cluster access) |
