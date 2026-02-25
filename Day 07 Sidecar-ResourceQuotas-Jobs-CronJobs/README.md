# 🧩 Kubernetes Sidecar Containers & Resource Quotas

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-24.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-Config-CC2222?style=for-the-badge&logo=yaml&logoColor=white)
![kubectl](https://img.shields.io/badge/kubectl-CLI-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![KOPS](https://img.shields.io/badge/KOPS-Cluster--Mgmt-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Namespace](https://img.shields.io/badge/Namespace-ResourceQuota-7B1FA2?style=for-the-badge&logo=kubernetes&logoColor=white)

---

## 📖 Introduction

In production Kubernetes, your main application container rarely runs alone. **Sidecar containers** share the same pod — same network, same volumes, same lifecycle — and handle cross-cutting concerns: dependency checks, log transformation, proxying, and real-time data enrichment. They keep application code focused while infrastructure concerns live in purpose-built containers beside it.

**Resource Quotas** are what stop a single namespace from consuming an entire cluster. Without them, one runaway deployment can starve every other workload on the node. In any multi-team environment, quotas are not optional — they're how SLAs stay intact.

This repository demos both concepts together: a multi-container pod with three init containers, an adapter sidecar, namespace-level quota enforcement, and a LimitRange that sets per-container CPU and memory floors.

---

## 🔧 Sidecar Container Patterns

### Init Containers

Run **sequentially before** main containers start. If any fails, the pod restarts. Use them for:
- Cloning git repos or fetching remote config before the app needs it
- Blocking until a dependency is reachable (DNS check, port probe)
- One-time setup tasks: schema migration, certificate generation

### Adapter Containers

Run **in parallel** with the main container, sharing a volume. They transform or enrich data in real time — in this repo, the adapter writes a new timestamped HTML line every 5 seconds into a shared volume that nginx serves immediately. No app code changes needed.

### Ambassador Containers

Act as outbound proxies for the main container. Envoy (Istio/Linkerd) is the canonical example — it intercepts traffic, handles retries, circuit breaking, and mTLS without the app knowing.

---

## 📦 Full Deployment YAML

One manifest creates everything: namespace, quota, limit range, multi-container deployment, and service.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    name: development

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: development
spec:
  hard:
    requests.cpu: "1000m"      # Total CPU requests cap across all pods
    limits.cpu: "2000m"        # Total CPU limits cap across all pods
    requests.memory: 1Gi       # Total memory requests allowed
    limits.memory: 2Gi         # Total memory limits allowed
    pods: "10"                 # Hard pod count ceiling for this namespace
    replicationcontrollers: "20"
    resourcequotas: "10"
    services: "5"              # Max 5 services — prevents uncontrolled exposure

---
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-memory-min-max-demo-lr
  namespace: development
spec:
  limits:
    - max:
        cpu: "200m"
        memory: "512Mi"
      min:
        cpu: "100m"            # Every container must request at least 100m CPU
        memory: 100Mi
      type: Container

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: realtime-production-deployment
  namespace: development
  labels:
    env: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      env: prod
  template:
    metadata:
      labels:
        env: prod
    spec:
      initContainers:

        # Step 1 — Clone web content into the shared volume before nginx starts
        - name: init-container
          image: alpine/git
          command: ["/bin/sh"]
          args:
            - "-c"
            - "git clone https://github.com/exampleuser/sidecar-test.git /html"
          volumeMounts:
            - name: shared-vol
              mountPath: /html/

        # Step 2 — Block until myservice DNS resolves (dependency gate)
        - name: wait-for-service
          image: busybox
          command:
            - "sh"
            - "-c"
            - "until nslookup myservice.development.svc.cluster.local;
               do echo waiting for myservice; sleep 2; done"

        # Step 3 — Write a status file confirming init tasks completed
        - name: perform-task
          image: busybox
          command:
            - "sh"
            - "-c"
            - 'echo "Init container tasks completed" > /tasks/status.txt'
          volumeMounts:
            - name: shared-vol
              mountPath: /tasks

      containers:

        # Adapter sidecar — appends live timestamps to index.html every 5s
        - name: adapter-container
          image: dummyrepo/kubegame:v1
          command: ["/bin/sh"]
          args:
            - "-c"
            - 'while true; do echo "<h1>$(date)</h1>" >> /html/index.html; sleep 5; done'
          ports:
            - containerPort: 80
          volumeMounts:
            - name: shared-vol
              mountPath: /html/
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "150m"
              memory: "350Mi"

        # Main container — nginx serves the shared volume
        - name: main-container
          image: dummyrepo/kubegame:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          volumeMounts:
            - name: shared-vol
              mountPath: /usr/share/nginx/html/
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "150m"
              memory: "350Mi"

      volumes:
        - name: shared-vol
          emptyDir: {}   # Ephemeral volume shared between all containers in the pod

---
apiVersion: v1
kind: Service
metadata:
  namespace: development
  name: myservice
  labels:
    env: prod
spec:
  selector:
    env: pod
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  type: NodePort
```

---

## 🚀 Getting Started

### Provision the cluster

```bash
kops create -f cluster.yml
```
> Registers the cluster definition from the YAML spec — does not provision infrastructure yet.

```bash
kops update cluster --name example.local --yes
```
> Provisions all AWS resources: EC2 nodes, VPC, Route53 records, IAM roles.

```bash
kops validate cluster --wait 10m
```
> Polls until all nodes join and the control plane is healthy, or fails after 10 minutes.

```bash
kops export kubeconfig --name example.local --admin
```
> Writes admin kubeconfig to `~/.kube/config` so `kubectl` can authenticate to the cluster.

```bash
kubectl cluster-info
```
> Confirms the API server and CoreDNS are reachable.

```bash
kubectl get nodes -o wide
```
> Lists all nodes with IPs, OS, and Kubernetes version — verifies the cluster is fully joined.

---

### Deploy the application

```bash
kubectl apply -f production.yaml
```
> Creates namespace, quota, limitrange, deployment, and service in one shot.

```bash
kubectl get pods
```
> Lists pods — you'll see `Init:0/3` cycling up to `Init:2/3` before the pod enters `Running`.

```bash
watch kubectl get pods -o wide
```
> Streams live pod status updates every 2 seconds — best way to watch the init container sequence play out.

---

### Switch namespace context

```bash
kubens development
```
> Sets `development` as the active namespace so you don't need `-n development` on every command.

```bash
kubens default
```
> Switches back to the default namespace.

---

### Inspect containers

```bash
kubectl exec -it app1-66f78b979-2jm4k -- bash
```
> Shells into the pod (default first container) — useful after readiness probe testing.

```bash
kubectl exec -it app1-66f78b979-jnxhg -- bash
```
> Shells into a second replica for cross-pod verification.

```bash
kubectl exec -it realtime-production-deployment-7d7f7d7d97-pjdtt -c adapter-container -- sh
```
> Targets the `adapter-container` sidecar specifically — check the shared volume content here.

```bash
kubectl exec -it realtime-production-deployment-7d7f7d7d97-pjdtt -c main-container -- sh
```
> Targets the `main-container` — verify nginx is serving the latest content from the shared volume.

---

### Logs

```bash
kubectl logs realtime-production-deployment-b68fcc9cc-fpdk8
```
> Prints stdout from the first container in the pod.

```bash
kubectl logs realtime-production-deployment-b68fcc9cc-fpdk8 wait-for-service -f | grep -i service
```
> Follows the `wait-for-service` init container log and filters for DNS resolution messages.

---

### Quota and namespace inspection

```bash
kubectl describe ns development
```
> Shows live quota usage vs hard limits — `Used` vs `Hard` for CPU, memory, pods, and services.

```bash
kubectl describe pod realtime-production-deployment-b68fcc9cc-fpdk8
```
> Full pod detail: init container states, resource requests, events, and scheduling decisions.

---

### Scale and expose

```bash
kubectl scale deployment realtime-production-deployment --replicas 6
```
> Scales to 6 replicas — if this breaches the `pods: "10"` quota, new pods are blocked with a clear error.

```bash
kubectl expose deployment realtime-production-deployment --name myservice --port 80 --target-port 80 --type NodePort
```
> Creates a NodePort Service routing external traffic to port 80 inside the pods.

```bash
kubectl get svc -o wide
```
> Lists services with cluster IPs, node ports, and selector labels.

---

### Tear down

```bash
kubectl delete -f production.yaml
```
> Removes all resources defined in the manifest: deployment, service, quota, namespace.

```bash
kops delete -f cluster.yml --yes
```
> Destroys the KOPS cluster and all AWS infrastructure — irreversible, use with care.

---

## 🎭 Practical Demo

### Init container ordering

Deploy and watch:

```bash
watch kubectl get pods -o wide
```

Pod status will cycle: `Init:0/3` → `Init:1/3` → `Init:2/3` → `Running`. The `wait-for-service` init container deliberately blocks until `myservice.development.svc.cluster.local` resolves — meaning the Service must exist before the Deployment's main containers start.

### Adapter sidecar in action

Once running, exec into the main container:

```bash
kubectl exec -it realtime-production-deployment-7d7f7d7d97-pjdtt -c main-container -- sh
# tail -f /usr/share/nginx/html/index.html
```

You'll see new `<h1>` timestamp lines appearing every 5 seconds — written by the adapter sidecar into the shared `emptyDir` volume, served immediately by nginx. Neither container modified the other; they communicate entirely through the shared volume.

### What happens when quota is exceeded

```bash
kubectl run test1 --image nginx:latest
kubectl run test2 --image nginx:latest
kubectl run test3 --image nginx:latest
```

Once the namespace hits `pods: "10"`, Kubernetes rejects the next pod with:

```
Error from server (Forbidden): pods "test3" is forbidden:
exceeded quota: object-counts, requested: pods=1, used: pods=10, limited: pods=10
```

Existing pods are completely unaffected. Check live quota state:

```bash
kubectl describe ns development
```

---

## 📊 Resource Units Quick Reference

| Unit | Meaning | Example |
|---|---|---|
| `m` | Millicores — 1/1000th of a CPU core | `100m` = 0.1 CPU |
| `Mi` | Mebibytes (1024² bytes) | `256Mi` ≈ 268 MB |
| `Gi` | Gibibytes (1024³ bytes) | `1Gi` ≈ 1.07 GB |
| `requests` | Guaranteed minimum — what the scheduler reserves on a node | `cpu: "100m"` |
| `limits` | Hard ceiling — container is CPU-throttled or OOM-killed above this | `memory: "350Mi"` |

---

## 🏁 Conclusion

Sidecar containers let you layer infrastructure concerns onto any application without touching its code. Init containers enforce strict dependency ordering — your main app never starts in a broken state. Adapter sidecars handle real-time data transformation and enrichment in parallel. Ambassador sidecars abstract outbound networking. Together they improve observability, integration, and reliability with zero coupling to application logic.

Resource Quotas are what make shared Kubernetes clusters stable at scale. Without them, one misconfigured deployment can exhaust cluster resources and cause cascading failures across unrelated teams. Pair Quotas with LimitRange to set per-container CPU and memory floors and ceilings, and you get predictable consumption, fair multi-tenant allocation, and a hard stop against runaway workloads.

Both patterns belong in every production Kubernetes environment.
