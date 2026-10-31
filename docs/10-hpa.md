# Lesson 10 — Horizontal Pod Autoscaler (HPA)

**HPA** = Horizontal Pod Autoscaler

## What problem does this solve?

Every deployment so far has a fixed `replicas` count. If traffic spikes, the
webapp stays at whatever replica count you hard-coded — it either falls over
under load, or you're paying for peak capacity 24/7 even when traffic is low.

HPA watches a metric (CPU, memory, or custom/external metrics) and adjusts
`spec.replicas` on a Deployment/StatefulSet automatically, between a min and
max bound you define. This is **horizontal** scaling (more pods), as opposed
to **vertical** scaling (bigger pods — that's VerticalPodAutoscaler, a
different, less commonly used object).

This only applies to the **webapp**, not Mongo. Mongo is a single stateful
instance backed by a `ReadWriteOnce` PVC — you can't just spin up a second
replica and have it share the same data directory. Scaling a real database
horizontally needs replication (a Mongo ReplicaSet), which is a separate,
much bigger topic outside this course's scope.

---

## Why HPA needs the Metrics Pipeline

HPA doesn't measure CPU/memory itself — it queries the **`metrics.k8s.io`**
API, which is served by the **Metrics Server** add-on. Without Metrics Server
installed, `kubectl top` and HPA both fail, because there is no CPU/memory
data source in the cluster at all.

```
┌──────────────┐   scrapes cAdvisor    ┌────────────────┐   metrics.k8s.io API   ┌─────────────────┐
│   kubelet    │ ─────────────────────►│  Metrics Server │◄──────────────────────│ HPA controller  │
│ (per node)   │                       │                 │                        │ (control plane) │
└──────────────┘                       └────────────────┘                        └─────────────────┘
                                                                                          │
                                                                                   patches replicas
                                                                                          ▼
                                                                                  ┌─────────────────┐
                                                                                  │   Deployment     │
                                                                                  └─────────────────┘
```

On Minikube, enable it with one command (see commands doc). In managed
clusters (EKS/GKE/AKS) Metrics Server is usually pre-installed or a one-line
add-on install as well.

---

## Why HPA needs `resources.requests` (from lesson 6)

CPU/memory `target.type: Utilization` is a **percentage of the container's
request**, not an absolute value and not the limit. The webapp's request from
lesson 6 is `cpu: 50m`. A target of `averageUtilization: 50` means: *scale up
once pods are, on average, using more than 25m CPU (50% of 50m)*.

If a container has no CPU request set, HPA cannot compute a percentage and
the metric shows `<unknown>` forever — one of the most common real-world HPA
failures (see "Common mistakes" below).

---

## The autoscaling/v2 API (current, not v1 or v2beta*)

- `autoscaling/v1` only supports a single CPU-utilization metric — legacy,
  avoid it.
- `autoscaling/v2beta1` / `v2beta2` are deprecated and removed in modern
  Kubernetes (v2beta2 removed in 1.26).
- `autoscaling/v2` has been GA since Kubernetes 1.23 and is what you should
  always use today. It supports:
  - Multiple metrics at once (CPU **and** memory, as in this lesson).
  - Custom metrics (e.g. requests-per-second via a Prometheus adapter —
    covered conceptually in lesson 11, monitoring).
  - External metrics (e.g. queue depth from a managed message broker).
  - **`behavior`** — fine-grained control over how fast to scale up/down
    (stabilization windows, percent/pod-count policies) — GA since 1.23,
    used in this lesson to scale up fast but scale down cautiously.
  - **`ContainerResource`** metric type (GA since 1.30) — targets a specific
    container's usage inside a multi-container pod, instead of averaging
    across all containers. Not needed here (single-container pods) but worth
    knowing if you later add sidecars.

---

## How the HPA control loop works

1. Every `--horizontal-pod-autoscaler-sync-period` (default 15s), the HPA
   controller queries current metric values for all pods matching
   `scaleTargetRef`'s selector.
2. It computes: `desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))`
3. If `desiredReplicas` differs from the current count (accounting for
   `behavior` stabilization/policies), it patches `spec.replicas` on the
   Deployment.
4. The Deployment controller then does its normal rolling update to reach
   that replica count.

HPA never edits Pod specs directly — it only ever changes the replica count
on the target's parent resource.

---

## `minReplicas` / `maxReplicas` / `behavior` design in this lesson

```yaml
minReplicas: 2
maxReplicas: 5
```
- `minReplicas: 2` — never scale below 2, for basic availability even at idle.
- `maxReplicas: 5` — hard ceiling so a runaway metric (or an attack) can't
  scale the Deployment into exhausting the cluster/node capacity.

```yaml
behavior:
  scaleUp:   { stabilizationWindowSeconds: 0,  ... }
  scaleDown: { stabilizationWindowSeconds: 60, ... }
```
- Scale up **fast** (no stabilization delay) — you want extra capacity the
  moment load appears.
- Scale down **slowly** (60s stabilization window, at most 1 pod removed per
  60s) — avoids "flapping": scaling down right before another burst arrives,
  then having to scale up again immediately.

This asymmetry (fast up, slow down) is the standard production pattern.

---

## What changes in this lesson

### New files

| File | Purpose |
|---|---|
| `lessons/10-hpa/webapp-hpa.yaml` | HorizontalPodAutoscaler targeting `webapp-deployment`, CPU + memory metrics, asymmetric scale-up/down behavior |
| `lessons/10-hpa/load-generator.yaml` | Throwaway busybox Pod that hammers `webapp-service` with parallel requests, used to trigger a scale-up for real |

No existing manifests need editing — HPA is a standalone object that
references `webapp-deployment` by name. The `resources.requests.cpu: 50m`
already set on `webapp-deployment` in lesson 6 is exactly what the HPA needs;
nothing else changes.

---

## Common mistakes

### 1. `TARGETS` column shows `<unknown>/50%` forever

Either:
- Metrics Server isn't installed/running (`kubectl top pods` also fails).
- The target container has no `resources.requests.cpu` set — HPA has no
  denominator to compute a percentage from.

### 2. HPA never scales down

Check `behavior.scaleDown.stabilizationWindowSeconds` — if it's very high
(or you're still within the window since the last scale-up), it's working
as designed, just being cautious. Also check `minReplicas` isn't already the
floor you're seeing.

### 3. Load test doesn't trigger scaling

- The pod might not actually be CPU-bound — a Node.js app returning a static
  response for `GET /` may use so little CPU that even the full parallel
  load-generator can't cross the 25m threshold. Increase the number of
  parallel loops, or lower the CPU request/target for the demo.
- `kubectl top pods` reports 0m/0Mi for a while right after pods start —
  Metrics Server needs one full scrape interval (~15-60s) before data appears.

### 4. Confusing `limits` with the HPA target

`averageUtilization` is always relative to **requests**, never `limits`.
Doubling the CPU `limit` does nothing to the HPA math.

### 5. Scaling a stateful workload horizontally without replication

Never point an HPA at a Deployment/StatefulSet backed by a single
`ReadWriteOnce` volume expecting more replicas to "just work" — only the
first pod can mount it; the rest will be stuck `Pending`. This is why only
`webapp-deployment` gets an HPA in this lesson.

---

## Debugging exercise

### Scenario: HPA stuck showing `<unknown>` targets

This simulates the single most common real-world HPA failure: someone
removes (or never sets) the CPU request on the target Deployment.

**Break it:**

1. Edit `lessons/06-resource-limits/webapp-deployment.yaml` (or your live
   cluster copy) and comment out/remove the `resources.requests.cpu` line,
   then re-apply it.
2. Apply the HPA if you haven't already:
   ```bash
   kubectl apply -f lessons/10-hpa/webapp-hpa.yaml
   ```
3. Watch it:
   ```bash
   kubectl get hpa webapp-hpa --watch
   ```

**Diagnose it:**

```bash
# TARGETS column shows <unknown>/50% instead of a percentage
kubectl get hpa webapp-hpa

# describe shows the actual reason in Conditions
kubectl describe hpa webapp-hpa
# Look for: "unable to get metrics for resource cpu: missing request for cpu"

# confirm the Deployment container has no CPU request
kubectl get deployment webapp-deployment -o jsonpath='{.spec.template.spec.containers[0].resources}'
```

**Fix it:**

Restore `resources.requests.cpu: "50m"` on the webapp container and
re-apply. Within one HPA sync interval (~15s), `TARGETS` should show a real
percentage again.

---

## Review questions

1. Why does an `averageUtilization: 50` target depend on `resources.requests`
   and not `resources.limits`? Walk through the actual percentage math with
   the webapp's `50m` CPU request.

2. Why wasn't an HPA added for the Mongo deployment in this lesson? What
   would need to change about how Mongo is deployed before it could be
   horizontally autoscaled?

3. What's the practical difference between `autoscaling/v1` and
   `autoscaling/v2`? Why does this lesson use `v2`?

4. In this lesson's `behavior` block, scale-up has `stabilizationWindowSeconds: 0`
   and scale-down has `60`. What problem does this asymmetry prevent, and
   what would happen if you swapped the values?

5. `kubectl top pods` and the HPA both show no data / `<unknown>` right
   after a fresh cluster start. What two independent things do you need to
   check to find out why?
