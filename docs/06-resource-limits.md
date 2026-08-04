# Lesson 6 — Resource Requests & Limits

## What problem does resource management solve?

Kubernetes runs many pods from many teams on shared nodes. Without resource
declarations, every pod competes for CPU and memory on a first-come,
first-served basis — the **noisy neighbour problem**:

- One memory-hungry pod grows until the node is pressured, and the kernel
  OOM-kills random processes.
- A CPU-hungry pod monopolises cores and starves adjacent pods.
- The scheduler places pods on nodes arbitrarily, with no awareness of what
  each container actually needs.

Resource `requests` and `limits` are your contract with Kubernetes about how
much a container needs and how much it is allowed to use.

---

## Requests vs Limits — the critical distinction

### Requests — "what this container is guaranteed"

- The scheduler reads requests when deciding **which node** to place the pod on.
  It only places the pod on a node that has at least this much unallocated
  capacity.
- Once running, the container is **guaranteed** this much resource — it will
  never be starved below its request.
- Requests affect **QoS class** (see below).

### Limits — "what this container is allowed to use"

- A hard ceiling enforced by the Linux kernel via cgroups.
- The container can burst above its request up to its limit, but never
  above its limit.
- **CPU limit** → the container is **throttled** (paused by the CFS scheduler)
  if it tries to use more. It keeps running, just more slowly.
- **Memory limit** → if the container exceeds this, it is **OOM-killed**
  immediately. Kubernetes then restarts it (if the restart policy allows).

```
Requests vs Limits visualised:
──────────────────────────────────────────────────────────────────────
  Node capacity:       ████████████████████████████████  (8 GiB RAM)
  Pod A request:       ██  (256Mi) — scheduler guarantees this
  Pod A actual usage:  ████ (400Mi) — fine, within limit
  Pod A limit:         ████████ (512Mi) — hard ceiling
                                         ^ exceed this → OOM kill
──────────────────────────────────────────────────────────────────────
```

---

## CPU units — millicores

CPU is measured in **millicores** (m). 1000m = 1 CPU core.

| Value | Meaning |
|---|---|
| `50m` | 0.05 cores — 5% of one core |
| `500m` | 0.5 cores — half a core |
| `1` or `1000m` | 1 full core |
| `2` | 2 full cores |

**CPU limit behaviour:** throttled, not killed. A container at its CPU limit
runs slowly but stays alive. This makes CPU starvation hard to detect — the
container appears healthy while being very slow.

## Memory units

| Value | Meaning |
|---|---|
| `64Mi` | 64 mebibytes (64 × 1024² bytes) — use `Mi` not `MB` |
| `256Mi` | 256 MiB |
| `1Gi` | 1 gibibyte |

**Memory limit behaviour:** OOM-killed immediately when exceeded. You will see
`OOMKilled` in the pod's `lastState` reason.

---

## QoS classes — a side effect of your request/limit settings

Kubernetes assigns every pod one of three **Quality of Service** classes based
purely on whether you set requests and limits. This class controls the
**eviction priority** when a node runs low on resources.

| Class | Rule | Eviction priority |
|---|---|---|
| **Guaranteed** | Every container has requests == limits for both CPU and memory | Last to be evicted |
| **Burstable** | At least one container has a request or limit set | Middle |
| **BestEffort** | No container has any requests or limits | First to be evicted |

Our pods after this lesson are **Burstable** (requests < limits). For a
production database you'd want **Guaranteed** (set `requests == limits` on
Mongo) to ensure it's never evicted under node pressure.

```bash
# See the QoS class of a running pod
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

---

## Namespace-level controls (preview)

Two objects let cluster admins enforce resource governance at the namespace
level. We don't implement them here but you should know they exist:

- **LimitRange**: sets default requests/limits for containers that don't specify
  their own, and enforces min/max bounds. Protects against pods with no limits.
- **ResourceQuota**: caps the total requests/limits across all pods in a namespace.
  Prevents one team from consuming all cluster resources.

---

## What changed in this lesson

### `lessons/06-resource-limits/webapp-deployment.yaml`

Added a `resources` block to the webapp container (probe config carried forward
from lesson 5):

```yaml
resources:
  requests:
    cpu: "50m"       # 0.05 cores — Node.js idle usage
    memory: "64Mi"   # ~30–40 MiB actual RSS + headroom
  limits:
    cpu: "200m"      # 0.2 cores max; throttle under heavy load
    memory: "128Mi"  # OOM-kill if exceeded
```

**Why these values?**  
The demo webapp is a tiny Node.js process. It idles at ~30–40 MiB RSS and uses
negligible CPU when there's no load. The limits give it 4× CPU headroom for
bursts and 2× memory headroom before an OOM kill.

### `lessons/06-resource-limits/mongo-deployment.yaml`

Added a `resources` block to the mongodb container (startup/liveness/readiness
probes carried forward from lesson 5):

```yaml
resources:
  requests:
    cpu: "100m"      # 0.1 cores — mongo needs a guaranteed slice for journal writes
    memory: "256Mi"  # mongo:8 baseline: WiredTiger cache + journal + overhead
  limits:
    cpu: "500m"      # 0.5 cores max
    memory: "512Mi"  # cap the WiredTiger cache growth in a dev cluster
```

**Why Mongo needs more memory than webapp?**  
MongoDB's WiredTiger storage engine maintains an in-memory cache. By default it
targets 50% of (total RAM − 1 GiB). On a 4 GiB Minikube node that would be
~1.5 GiB. The memory limit caps it at 512 MiB for the dev cluster.

---

## How to apply

### Step 1 — Apply the base stack (if fresh cluster)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
```

### Step 2 — Apply lesson 6 deployments

```bash
kubectl apply -f lessons/06-resource-limits/mongo-deployment.yaml
kubectl apply -f lessons/06-resource-limits/webapp-deployment.yaml
```

### Step 3 — Verify resources were set

```bash
kubectl describe pod -l app=webapp | grep -A 6 "Limits\|Requests"
kubectl describe pod -l app=mongo  | grep -A 6 "Limits\|Requests"
```

Expected output for webapp:
```
Limits:
  cpu:     200m
  memory:  128Mi
Requests:
  cpu:     50m
  memory:  64Mi
```

### Step 4 — Check the QoS class

```bash
kubectl get pod -l app=webapp -o jsonpath='{.items[0].status.qosClass}{"\n"}'
kubectl get pod -l app=mongo  -o jsonpath='{.items[0].status.qosClass}{"\n"}'
```

Both should print `Burstable`.

### Step 5 — Check node resource allocation

```bash
kubectl describe node minikube | grep -A 8 "Allocated resources"
```

This shows how much of the node's total CPU and memory is allocated across all
running pods. Your new requests will appear here.

---

## Common mistakes

### 1. No requests or limits at all (BestEffort)

Pods without any resources are the first to be evicted when the node is under
pressure. In production this is never acceptable — always set at least requests.

### 2. Limits without requests

```yaml
resources:
  limits:
    memory: "256Mi"
```

Kubernetes treats the request as equal to the limit in this case — the pod
becomes Guaranteed. This is often unintentional and over-allocates reserved
capacity on the node.

### 3. Memory limit lower than the application's startup footprint

A common mistake when first adding limits: the container starts, loads its
runtime, and is immediately OOM-killed before it does anything useful. The pod
enters CrashLoopBackOff with reason `OOMKilled`.

### 4. CPU limit too low — silent throttling

Setting `cpu: "10m"` on an app that needs 100m during initialisation causes
severe throttling. The container appears Running and passes health probes
eventually, but everything is 10× slower than expected. Check
`kubectl top pod` (requires metrics-server) to see actual CPU usage.

### 5. Setting requests == limits on everything blindly

While Guaranteed QoS sounds appealing, it means the scheduler must find a node
with exactly that much free capacity reserved for every container. In a cluster
with many pods, this fragments resources and leads to pods pending even when
there's plenty of total capacity available.

---

## Debugging exercise

### Scenario: memory limit too low → OOMKilled crash loop

1. Edit `lessons/06-resource-limits/webapp-deployment.yaml`. Change the memory
   limit from `128Mi` to `10Mi`:
   ```yaml
   limits:
     memory: "10Mi"
   ```

2. Apply:
   ```bash
   kubectl apply -f lessons/06-resource-limits/webapp-deployment.yaml
   ```

3. Watch:
   ```bash
   kubectl get pods --watch
   ```
   The new webapp pod will start, get OOM-killed almost immediately, and enter
   `CrashLoopBackOff`. The `RESTARTS` counter climbs.

4. Diagnose:
   ```bash
   # Step 1 — see the exit reason on the pod
   kubectl describe pod -l app=webapp
   # Look for: Last State: Terminated  Reason: OOMKilled

   # Step 2 — confirm current resource settings
   kubectl get pod -l app=webapp -o jsonpath=\
     '{.items[0].spec.containers[0].resources}' | python3 -m json.tool
   ```

5. Fix: revert the memory limit back to `128Mi` and re-apply. The pod should
   stabilise and `RESTARTS` should stop incrementing.

---

## Review questions

1. What is the difference between a resource **request** and a resource
   **limit**? Which one does the Kubernetes scheduler use, and why?

2. A container has `cpu limit: 100m` and is trying to use 400m. What
   happens? Is the container killed? How is this different from exceeding
   the memory limit?

3. You set `requests.memory: 512Mi` and `limits.memory: 512Mi` on every
   container in a pod. What QoS class does the pod get, and what does that
   mean for eviction under node pressure?

4. Your pod keeps restarting with `Reason: OOMKilled`. Walk through the
   steps you'd take to investigate and fix it.

5. A colleague says "I'll just not set any resource limits — why constrain
   the app?" What are the two main risks of running pods without limits in a
   shared cluster?
