# 🩺 Kubernetes Probes Deep Dive — Readiness · Liveness · Startup

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-24.0-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![YAML](https://img.shields.io/badge/YAML-Config-CC2222?style=for-the-badge&logo=yaml&logoColor=white)
![kubectl](https://img.shields.io/badge/kubectl-CLI-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-Production--Grade-0A66C2?style=for-the-badge&logo=azuredevops&logoColor=white)
![nginx](https://img.shields.io/badge/nginx-Webserver-009639?style=for-the-badge&logo=nginx&logoColor=white)

---

## 📖 Introduction

Production systems fail in ways you never anticipated during development. A container starts, passes its initial health check, and gets traffic — but internally, it's deadlocked, half-initialised, or waiting on a database connection that never came back. Users hit 502s. Your on-call phone rings at 2 AM.

Kubernetes Probes exist precisely to prevent that scenario.

A **probe** is a diagnostic check that the `kubelet` runs against a running container on a defined schedule. Depending on the probe type and result, Kubernetes either stops routing traffic to the pod, restarts it, or holds off on activating other probes until the application is genuinely ready. When tuned correctly, probes give your cluster the intelligence to detect and self-heal from failure states — automatically, without a human in the loop.

This repository demonstrates all three probe types using a real containerised web application (`rayeez/kubegame:v2`) deployed as a 3-replica Kubernetes Deployment. You'll see the full workflow — from generating the deployment manifest with `--dry-run`, patching in probes, applying to the cluster, and verifying behaviour from inside running pods.

---

## 📂 Repository Structure

```
k8s-probes-deep-dive/
├── readiness-deployment.yaml       # Deployment with Readiness Probe only
├── readiness-liveness-deployment.yaml  # Deployment with both Readiness + Liveness Probes
├── commands_used.txt
└── README.md
```

---

## 🔬 What Are Kubernetes Probes?

Kubernetes does not know what your application does internally. It only knows whether a container process is running. Without probes, a pod sitting in `Running` state could be serving 500 errors, deadlocked, or stuck waiting on config that never loaded — and Kubernetes would happily keep routing traffic to it.

Probes solve this by giving `kubelet` a window into the actual health of your application. There are three:

| Probe | Question It Answers |
|---|---|
| **Readiness** | Is this container ready to accept traffic right now? |
| **Liveness** | Is this container still alive and functional? |
| **Startup** | Has this container finished its slow initialisation? |

Each probe uses one of three mechanisms — `httpGet`, `tcpSocket`, or `exec` — to perform its check. The kubelet executes the check on the defined schedule and acts on the result accordingly.

---

## 🤔 Why Use Probes in Production?

The simple answer: without probes, you are flying blind.

Concretely, probes let you:

- **Prevent traffic routing to unready containers** — A freshly scheduled pod that hasn't finished warming up won't receive a single request until the readiness probe passes.
- **Automatically recover from stuck processes** — Applications deadlock. Memory corruption happens. A liveness probe catches these states and triggers a restart before your users notice.
- **Protect slow-starting legacy apps** — Startup probes buy slow applications the time to initialise without getting killed by an overeager liveness check.
- **Improve SLA and uptime** — Probes are the foundational building block of self-healing infrastructure. They are what makes "it just restarts itself" actually true.

In any environment running real user traffic, operating without probes is a reliability risk you should not accept.

---

## ✅ Phase 1 — Readiness Probe

### What it does

A readiness probe tells Kubernetes whether a container is ready to serve traffic. When the probe fails, the pod is **removed from the Service's endpoints** — no traffic reaches it. Critically, the container is **not restarted**. The pod keeps running; it simply stops receiving requests until the probe passes again.

This is the right tool for:
- Applications that need time to warm up after scheduling
- Pods that temporarily become unavailable (cache reload, dependency temporarily down)
- Controlled rollouts where you want each replica verified before the next is updated

### Step 1 — Generate the Base Manifest with dry-run

Rather than writing the YAML from scratch, generate a base manifest and pipe it straight into the cluster:

```bash
kubectl create deployment app1 --image rayeez/kubegame:v2 --dry-run=client -o yaml
```
> Generates a Deployment manifest for `rayeez/kubegame:v2` without applying it — output goes to stdout for review or editing.

### Step 2 — Apply the Readiness Probe Deployment

```bash
echo 'apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app1
  name: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - image: rayeez/kubegame:v2
          name: kubegame
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10' | kubectl apply -f -
```
> Applies the inline Deployment YAML directly to the cluster by piping the manifest into `kubectl apply`.

### Readiness Probe — Full YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app1
  name: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - image: rayeez/kubegame:v2
          name: kubegame
          readinessProbe:
            httpGet:
              path: /index.html   # Checks that the main page is being served correctly
              port: 80            # nginx inside the container listens on port 80
            initialDelaySeconds: 5  # Wait 5s after container start before the first check
            periodSeconds: 10       # Re-run the check every 10 seconds thereafter
```

### Field Reference

| Field | Purpose |
|---|---|
| `httpGet.path` | The URL path the kubelet requests — a 2xx response means the pod is ready |
| `httpGet.port` | Port to target — must match what your container is actually listening on |
| `initialDelaySeconds` | Grace period before the first probe fires; set this based on your measured startup time |
| `periodSeconds` | Probe frequency — every 10 seconds is a sensible default for most apps |

### Step 3 — Verify Pods Are Ready

```bash
kubectl get pods
```
> Lists all pods with their `READY` status, `STATUS`, and `RESTARTS` count — readiness probe failures show as `0/1` in the READY column.

### Step 4 — Exec into Pods to Verify Probe Behaviour

Exec into a pod and manually simulate what the readiness probe does — confirm the target path is actually reachable from inside the container:

```bash
kubectl exec -it app1-66f78b979-2jm4k -- bash
```
> Opens an interactive bash shell inside pod `app1-66f78b979-2jm4k` to inspect the container filesystem and test endpoints manually.

```bash
kubectl exec -it app1-66f78b979-jnxhg -- bash
```
> Opens a shell in a second replica to verify consistent behaviour across all running pods in the Deployment.

Once inside, you can run:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:80/index.html
```
> Hits the readiness probe endpoint directly — a `200` response confirms the probe will pass; anything else explains why the pod is marked unready.

---

## 💓 Phase 2 — Adding Liveness Probe

### What it does

A liveness probe answers a different question: is this container still doing useful work, or has it entered a broken state it can't recover from on its own?

When a liveness probe fails consecutively beyond `failureThreshold`, `kubelet` **restarts the container**. This is the automatic recovery mechanism — deadlocks, OOM states, infinite loops, corrupted internal state. The liveness probe catches them and gives the container a clean slate.

Do not conflate liveness with readiness. A pod can be alive (liveness passes) but not ready (readiness fails) — this is normal and expected during startup or when waiting on a dependency. They serve different purposes and are evaluated independently.

### Step 5 — Apply the Combined Readiness + Liveness Deployment

```bash
echo 'apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app1
  name: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - image: rayeez/kubegame:v2
          name: kubegame
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20' | kubectl apply -f -
```
> Updates the existing `app1` Deployment in-place to add a liveness probe alongside the readiness probe — Kubernetes performs a rolling update across the 3 replicas.

### Readiness + Liveness — Full YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app1
  name: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - image: rayeez/kubegame:v2
          name: kubegame
          readinessProbe:
            httpGet:
              path: /index.html   # Readiness check — gates traffic into this pod
              port: 80
            initialDelaySeconds: 5   # First readiness check fires 5s after container start
            periodSeconds: 10        # Rechecks every 10 seconds

          livenessProbe:
            httpGet:
              path: /index.html   # Liveness check — detects broken/stuck container state
              port: 80
            initialDelaySeconds: 15  # Fires 15s after start — after readiness has already run
            periodSeconds: 20        # Rechecks every 20 seconds (less frequent than readiness)
```

### Why `initialDelaySeconds: 15` for Liveness?

The liveness probe is intentionally set to fire 10 seconds **after** the readiness probe. This ordering matters:

1. At `t=5s` — readiness probe runs for the first time. Pod may still be warming up; if it fails, it's simply held out of rotation. No restart.
2. At `t=15s` — liveness probe fires for the first time. By this point the app should be up. If liveness starts failing here, it genuinely indicates a broken state — and a restart is appropriate.

Setting both to the same `initialDelaySeconds` risks the liveness probe killing a container that's still starting up normally.

### Step 6 — Verify the Rolling Update

```bash
kubectl get pods
```
> After applying the updated manifest, watch pods cycle through `Terminating` → `ContainerCreating` → `Running` — confirms the rolling update completed successfully with the new probe config active.

### Step 7 — Exec into the Updated Pods

The pod names change after a rolling update. Exec into the new replicas to confirm both probes are reachable:

```bash
kubectl exec -it app1-5846c9b899-gkkt4 -- bash
```
> Opens a shell in the first updated replica to verify the container is healthy and both probe endpoints are reachable.

```bash
kubectl exec -it app1-5846c9b899-jmjzf -- bash
```
> Opens a shell in the second updated replica — cross-check that all pods in the Deployment behave identically.

```bash
kubectl exec -it app1-5846c9b899-xtmg7 -- bash
```
> Opens a shell in the third replica — validates that the rolling update applied cleanly across all three pods.

---

## 🚀 Startup Probe — For Slow-Starting Applications

### What it does

Startup probes solve one specific problem: applications with long initialisation times that would be killed by the liveness probe before they finish starting.

The startup probe runs **exclusively** — liveness and readiness are both disabled until it passes. Once it succeeds, kubelet hands control to the other two probes. If the startup probe never succeeds within `failureThreshold × periodSeconds`, the container is killed.

This is the pattern for JVM services, .NET apps, anything loading large ML models, or legacy applications with multi-minute startup sequences.

### YAML Example

```yaml
startupProbe:
  httpGet:
    path: /index.html
    port: 80
  failureThreshold: 30     # 30 attempts × 10s each = up to 5 minutes to start
  periodSeconds: 10        # Checks every 10 seconds during the startup window
```

With this config, the app gets up to 5 minutes to pass a single startup probe check. Once it does, readiness and liveness take over with their own schedules. Without this, you'd need to bloat `initialDelaySeconds` on your liveness probe to 300 seconds — meaning a real deadlock would go undetected for 5 minutes. Startup probes solve that cleanly.

---

## 🏗 Combined — All Three Probes Together

For a complete production deployment, all three probes work in sequence:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
  labels:
    app: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
        - name: kubegame
          image: rayeez/kubegame:v2

          # --- Startup Probe (runs first, blocks readiness + liveness) ---
          startupProbe:
            httpGet:
              path: /index.html
              port: 80
            failureThreshold: 30   # Up to 300s for the app to start
            periodSeconds: 10

          # --- Readiness Probe (gates traffic) ---
          readinessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10

          # --- Liveness Probe (triggers restarts on broken state) ---
          livenessProbe:
            httpGet:
              path: /index.html
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
```

### Traffic Flow

```
User Request
    │
    ▼
Kubernetes Service (app1)
    │
    ├─ Pod 1  [Startup: PASS → Readiness: PASS] ✅  ← Receives traffic
    ├─ Pod 2  [Startup: PASS → Readiness: FAIL] ❌  ← Removed from endpoints
    └─ Pod 3  [Startup: PASS → Readiness: PASS] ✅  ← Receives traffic

                                        │
                          If Liveness fails on Pod 2:
                                        ▼
                          kubelet restarts container → Pod 2 re-enters
                          startup → readiness cycle automatically
```

---

## 🎭 Practical Scenario

Picture this: your `kubegame` web app is deployed with 3 replicas behind a Service. During a rolling deploy:

**Without readiness probe:** New pods spin up and are immediately added to the Service endpoints. If nginx inside the container takes even 3–4 seconds to fully initialise, those first requests land on a container that hasn't finished starting — users get connection refused or a blank page.

**Without liveness probe:** Three days later, a pod gets into a state where nginx is running but returning 500 on every request due to a corrupted temp file. The process hasn't exited, so Kubernetes considers it healthy. The pod sits there silently failing. A third of your traffic is going to a broken replica.

**Without startup probe:** You're deploying a heavier variant of the app that takes 45 seconds to start. Your liveness probe has `initialDelaySeconds: 30`. At `t=30s` the liveness probe fires, gets a 503 (still starting), and kills the container. It starts again. Gets killed again. CrashLoopBackOff. Infinite loop that could have been avoided.

**With all three probes:**
- Startup probe gives the container a 300s window — more than enough
- Readiness probe holds the pod out of the Service until `/index.html` returns 200
- Liveness probe detects the nginx failure and restarts the container within ~60 seconds
- The bad pod is replaced cleanly; zero manual intervention, zero pager alerts

---

## ⌨️ Complete Command Reference

### Generate a base manifest without applying it

```bash
kubectl create deployment app1 --image rayeez/kubegame:v2 --dry-run=client -o yaml
```
> Outputs a ready-to-edit Deployment YAML to stdout — the starting point for adding probe configuration before applying.

---

### Apply the Readiness-only Deployment

```bash
echo '...' | kubectl apply -f -
```
> Applies a manifest piped from stdin directly into the cluster — no need to create a file on disk first.

---

### Check pod status after applying

```bash
kubectl get pods
```
> Lists all pods in the current namespace showing READY state, STATUS, RESTARTS, and AGE — your quick health check after every apply.

---

### Exec into pods to manually verify probe endpoints

```bash
kubectl exec -it app1-66f78b979-2jm4k -- bash
```
> Drops into an interactive shell inside the first replica of the readiness-only Deployment for manual endpoint testing.

```bash
kubectl exec -it app1-66f78b979-jnxhg -- bash
```
> Drops into the second replica — verify probe endpoint consistency across all pods in the Deployment.

---

### Apply the updated Readiness + Liveness Deployment

```bash
echo '...' | kubectl apply -f -
```
> Re-applies the Deployment manifest with the liveness probe added — Kubernetes performs a rolling update automatically.

---

### Exec into the updated pod replicas

```bash
kubectl exec -it app1-5846c9b899-gkkt4 -- bash
```
> Accesses the first updated replica (post-rolling-update) to confirm both readiness and liveness probe paths are reachable.

```bash
kubectl exec -it app1-5846c9b899-jmjzf -- bash
```
> Accesses the second updated replica for cross-replica verification.

```bash
kubectl exec -it app1-5846c9b899-xtmg7 -- bash
```
> Accesses the third updated replica — ensures the rolling update applied cleanly to every pod.

---

### Watch pods in real time

```bash
kubectl get pods -w
```
> Streams live pod status changes to the terminal — use this while applying to watch replicas cycle through readiness states.

---

### Inspect probe configuration and failure events

```bash
kubectl describe pod <pod-name>
```
> Dumps full pod detail including Liveness/Readiness/Startup probe config, container state, and the Events section showing any probe failures.

```bash
kubectl describe pod <pod-name> | grep -A 20 "Liveness\|Readiness\|Startup"
```
> Filters `describe` output to show only the configured probe parameters — quick way to confirm the right values are active.

```bash
kubectl describe pod <pod-name> | grep -A 30 Events
```
> Shows the Events block where probe failures are logged explicitly with timestamps and failure reasons.

---

### Check logs

```bash
kubectl logs <pod-name>
```
> Prints the container's stdout/stderr — check this if your probe path is returning unexpected status codes.

```bash
kubectl logs <pod-name> --previous
```
> Retrieves logs from the **previous** container instance — the only way to see what happened before a liveness-triggered restart killed it.

---

### Check Service endpoints

```bash
kubectl get endpoints
```
> Lists which pod IPs are currently in each Service's endpoint pool — only pods with passing readiness probes appear here.

---

### Monitor restart count

```bash
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].restartCount}'
```
> Outputs the exact restart count for a pod's container — a number that keeps climbing confirms repeated liveness probe failures.

---

## 🔧 Troubleshooting

When things don't behave as expected, work through this sequence:

**Step 1 — Read pod status**

```bash
kubectl get pods
```

`0/1 Running` = readiness probe failing. Container is running but not yet serving traffic. High `RESTARTS` = liveness probe triggering repeatedly.

**Step 2 — Dig into Events**

```bash
kubectl describe pod <pod-name>
```

The `Events` section at the bottom tells you exactly what's happening:

```
Warning  Unhealthy  8s    kubelet  Readiness probe failed: HTTP probe failed with statuscode: 503
Warning  Unhealthy  28s   kubelet  Liveness probe failed: Get "http://10.0.0.4:80/index.html": context deadline exceeded (Client.Timeout exceeded)
Normal   Killing    28s   kubelet  Container kubegame failed liveness probe, will be restarted
```

**Step 3 — Test the probe path from inside the pod**

```bash
kubectl exec -it <pod-name> -- bash
curl -v http://localhost:80/index.html
```
> If `curl` fails from inside the pod, the application is the problem — not the probe configuration.

**Step 4 — Check previous container logs after a restart**

```bash
kubectl logs <pod-name> --previous
```

After a liveness-triggered restart, the current container is fresh and its logs may be empty. `--previous` retrieves the logs from the container that was just killed — this is where the failure context lives.

**Step 5 — Verify endpoint membership**

```bash
kubectl get endpoints
```

If your pod is `1/1 Running` but not receiving traffic, check whether its IP appears in the endpoints list. If not, the readiness probe is silently failing on a schedule.

---

## 📊 Probe Comparison

| Probe Type | Primary Purpose | Restarts Pod? | Removes from Service? | Best Use Case |
|---|---|---|---|---|
| **Readiness** | Gate traffic until app is genuinely ready | ❌ No | ✅ Yes | Slow startup, temporary dependency unavailability |
| **Liveness** | Detect and recover from unrecoverable states | ✅ Yes | ✅ Yes (while restarting) | Deadlocks, infinite loops, corrupted state |
| **Startup** | Protect slow-starting apps from premature liveness failures | ✅ Yes (if never passes) | ❌ No | JVM apps, legacy services, large model loading |

---

## 🏭 Production Best Practices

**Always define readiness and liveness on every Deployment.** A workload without probes is trusting that your application never enters a degraded state. It will.

**Set `initialDelaySeconds` based on measured startup time, not guesswork.** Run your app under realistic load in staging, time how long it takes to be ready, and add a 20–30% buffer. Guessing too low causes false restarts; guessing too high delays failure detection.

**Don't make probes too aggressive.** A `periodSeconds: 2` liveness probe creates constant load and generates false positives under CPU spikes. Start at 10–20 seconds for liveness, 5–10 for readiness, and tighten only if your SLA demands it.

**Use startup probes for any app with JVM, .NET CLR, or slow framework initialisation.** These runtimes regularly take 30–90 seconds to reach a ready state. Don't work around it with inflated `initialDelaySeconds` — use the right tool.

**Keep liveness endpoints lightweight — don't check external dependencies there.** If your liveness probe queries the database and the database is temporarily down, all your pods restart simultaneously. Liveness should reflect only the internal state of the process. Readiness can (and should) check downstream dependencies because it controls traffic, not pod lifecycle.

**Test probe failure paths before going to production.** Kill the `/index.html` path and confirm the pod leaves the Service endpoints. Block the liveness path and watch the restart counter climb. If you haven't deliberately triggered failure in staging, you don't actually know your probes work.

---

## 🏁 Conclusion

Running containers in Kubernetes without probes is like running a load balancer without health checks — technically functional, practically dangerous. Kubernetes knows when your containers crash. Probes are what teach it to understand when your containers are broken without crashing.

The workflow in this repository demonstrates the complete progression: start with a dry-run to generate a clean base manifest, layer in a readiness probe to protect traffic routing, then add a liveness probe to enable automatic recovery from unrecoverable states. Each step is verified by exec-ing into running pods to confirm behaviour from the inside — not just trusting that the status column looks right.

In production, every Deployment should have at minimum a readiness probe. Add liveness for any workload that can deadlock or accumulate state corruption over time. Add startup for anything that needs more than 30 seconds to initialise. Tune the thresholds to your actual application, test the failure paths, and your cluster will handle outages that previously required a 2 AM intervention — silently, automatically, before users notice.

That's what self-healing infrastructure actually looks like.
