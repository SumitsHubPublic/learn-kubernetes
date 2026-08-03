# Lesson 5 — Health Probes (liveness / readiness / startup)

## What problem do health probes solve?

Without probes, Kubernetes only knows one thing about a container: whether its
process is still running. That is a very low bar. Consider these real scenarios:

| Scenario | Process running? | App actually healthy? |
|---|---|---|
| Node.js event loop deadlocked | Yes | No |
| App started but still loading config/DB connection | Yes | Not yet |
| Mongo fully up but waiting for journal replay on restart | Yes | Not yet |
| OOM killer about to fire | Yes (briefly) | No |

**Health probes** give you a way to tell Kubernetes what "healthy" means for
your specific container, so it can take action (restart or stop routing traffic)
rather than blindly assuming all is well.

---

## The three probe types

Understanding which probe to use is the most important thing in this lesson.
They are not interchangeable.

### Liveness probe — "is this container still alive?"

- **What happens on failure:** Kubernetes kills and restarts the container.
- **Use it for:** detecting a container that has become permanently broken —
  deadlock, infinite loop, full memory corruption — where a restart is the
  only fix.
- **Do NOT use it too aggressively.** A liveness probe that fires during
  normal slow operations (cold start, GC pause, legitimate burst) causes
  unnecessary restarts that make things worse. When in doubt, be lenient
  (high `failureThreshold`, generous `timeoutSeconds`).

### Readiness probe — "is this container ready to serve traffic?"

- **What happens on failure:** Kubernetes removes the pod from the Service's
  endpoint list. Traffic stops going to it. The container is NOT restarted.
  When the probe passes again, the pod is added back.
- **Use it for:** the period before an app is initialised (DB connections,
  cache warming, config loading), and for temporary overload conditions where
  the pod needs to shed traffic without dying.
- **Key insight:** a pod can be "alive" (liveness passes) but "not ready"
  (readiness fails). That is the correct state during a rolling update —
  the new pod isn't sent traffic until it's ready, so there's no downtime.

### Startup probe — "has this container finished starting up?"

- **What happens while it's running:** liveness and readiness probes are
  **suspended**. The container gets uninterrupted time to start.
- **What happens on failure after `failureThreshold` attempts:** the container
  is killed and restarted (same as a failed liveness).
- **Use it for:** containers with slow, variable startup times (databases,
  JVM apps, anything that replays a journal or loads large data on boot).
  Without a startup probe, you'd have to set a very high
  `initialDelaySeconds` on the liveness probe — which means a broken
  long-running container could go undetected for minutes.

```
Timeline for Mongo with startup probe:
─────────────────────────────────────────────────────────────────────
  0s   Container starts
  5s   First startup probe check (initialDelaySeconds: 5)
 10s   Second startup probe check ... eventually passes
 ──── startup probe PASSES ────
       Liveness probe starts (periodSeconds: 10)
       Readiness probe starts (periodSeconds: 10)
─────────────────────────────────────────────────────────────────────
```

---

## The three probe mechanisms

| Mechanism | How it works | Healthy condition |
|---|---|---|
| `httpGet` | K8s sends an HTTP GET to `path:port` | Response status 200–399 |
| `exec` | K8s runs a command inside the container | Exit code 0 |
| `tcpSocket` | K8s opens a TCP connection to a port | Connection succeeds |
| `grpc` | K8s sends a gRPC health check (1.24+) | gRPC status `SERVING` |

Use `httpGet` for HTTP services — it's the most descriptive.
Use `exec` for databases and binaries that don't speak HTTP (mongo, redis).
Use `tcpSocket` as a minimal check for TCP services where you can't run a command.

---

## Probe configuration parameters

Every probe shares these fields:

| Field | Default | Meaning |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait this many seconds after container start before the first check. |
| `periodSeconds` | 10 | How often to run the check. |
| `timeoutSeconds` | 1 | Each individual check must complete within this time. |
| `successThreshold` | 1 | How many consecutive successes to consider the probe passing (readiness can use >1). |
| `failureThreshold` | 3 | How many consecutive failures before taking action. |

**Total time before action = `failureThreshold` × `periodSeconds`**

For the startup probe: total budget = `failureThreshold` × `periodSeconds` +
`initialDelaySeconds`. Make this larger than your slowest observed startup.

---

## Which probes to add where — decision guide

```
Does the container serve HTTP?
  Yes → use httpGet for both liveness and readiness
  No  → use exec (run a health-check CLI command)

Is startup time slow or variable (>15–20s)?
  Yes → add a startup probe; set failureThreshold × periodSeconds ≥ worst-case start
  No  → just use initialDelaySeconds on liveness/readiness

Should temporary unresponsiveness stop traffic without restarting the container?
  Yes → use readiness probe (NOT liveness) for that condition
```

---

## What changed in this lesson

### `lessons/05-health-probes/webapp-deployment.yaml`

Added `readinessProbe` and `livenessProbe` to the webapp container.

The webapp is a Node.js HTTP server on port 3000. A GET `/` returns 200, so
that is a cheap, correct health endpoint:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 5    # Node.js starts fast; 5s is enough
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 3

livenessProbe:
  httpGet:
    path: /
    port: 3000
  initialDelaySeconds: 10   # slightly longer grace — don't restart a slow-starting pod
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 3
```

Why different `initialDelaySeconds` for the two?  
The readiness probe can safely fire early — if it fails, no traffic goes to the
pod, which is exactly what we want while the app is still initialising. The
liveness probe fires later because a false positive (restarting a healthy pod)
is disruptive.

### `lessons/05-health-probes/mongo-deployment.yaml`

Added `startupProbe`, `livenessProbe`, and `readinessProbe` to the mongodb
container.

Mongo does not serve HTTP, so we use an `exec` probe running `mongosh`:

```yaml
startupProbe:
  exec:
    command: [mongosh, --eval, "db.adminCommand('ping')", --quiet]
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 12     # 5 + 12×5 = 65s total budget for first boot

livenessProbe:
  exec:
    command: [mongosh, --eval, "db.adminCommand('ping')", --quiet]
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  exec:
    command: [mongosh, --eval, "db.adminCommand('ping')", --quiet]
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
```

`db.adminCommand('ping')` returns `{ ok: 1 }` and does **not** require
authentication in MongoDB 5+ — it is specifically designed to be a safe health
check even when a user is not logged in. This avoids embedding credentials in
the probe command (which would be visible in `kubectl describe pod`).

Why a startup probe for Mongo but not webapp?  
On first boot, `mongo:8` creates the `admin` user, sets up the journal, and
replays the oplog. This can take 15–30 seconds on a slow volume. Without a
startup probe, the liveness probe would fire during this window and restart
the container before it ever finishes initialising — a restart loop with no
crash, just bad timing.

---

## How to apply

### Step 1 — Apply the base stack (secrets, configmap, services)

These haven't changed from lesson 1, so just re-apply if the cluster is fresh:

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
```

Apply the services (unchanged):
```bash
# Mongo service — unchanged from lesson 1
kubectl apply -f lessons/01-k8s-basics/mongo.yaml   # only the Service part matters
# Or apply both at once (Deployment will be replaced below)
```

### Step 2 — Apply the lesson 5 deployments

```bash
kubectl apply -f lessons/05-health-probes/mongo-deployment.yaml
kubectl apply -f lessons/05-health-probes/webapp-deployment.yaml
```

### Step 3 — Watch the pods come up

```bash
kubectl get pods --watch
```

You should see the webapp pod go through:
```
NAME                                 READY   STATUS              RESTARTS
mongo-deployment-xxx                 0/1     Running             0        ← startup probe running
webapp-deployment-xxx                0/1     Running             0        ← readiness probe pending
mongo-deployment-xxx                 1/1     Running             0        ← startup → liveness/readiness pass
webapp-deployment-xxx                1/1     Running             0        ← readiness passes
```

Press Ctrl+C once both are `1/1 Running`.

### Step 4 — Inspect the probe configuration on a live pod

```bash
kubectl describe pod -l app=webapp | grep -A 20 "Liveness\|Readiness"
kubectl describe pod -l app=mongo  | grep -A 20 "Liveness\|Readiness\|Startup"
```

This shows Kubernetes's view of the probes — useful to confirm they were
applied correctly.

### Step 5 — Watch probe events in real time (optional)

```bash
kubectl get events --sort-by='.lastTimestamp' --watch
```

You will see `Unhealthy` events when a probe fails, and `Killing` events when
Kubernetes decides to restart a container.

---

## Common mistakes

### 1. Liveness probe too aggressive → restart loop

Setting `failureThreshold: 1` or `timeoutSeconds: 1` on a liveness probe for
a JVM or database container. A single slow GC pause or startup delay triggers
a restart, which triggers another startup delay, which triggers another restart.

**Rule:** liveness probes should only fire on conditions that a restart will
actually fix. Use `readiness` for "temporarily not ready".

### 2. No startup probe on a slow container, large `initialDelaySeconds` instead

```yaml
# Bad pattern for Mongo:
livenessProbe:
  initialDelaySeconds: 60   # blindly wait 60s before ANY liveness check
```

This means if the container gets stuck after the 60s delay, you won't detect
it for another `failureThreshold × periodSeconds` seconds on top.

```yaml
# Better: startup probe gates liveness precisely
startupProbe:
  failureThreshold: 12
  periodSeconds: 5           # starts checking at 5s, gives up at 65s
```

### 3. Using liveness where readiness is correct

A common mistake: using a liveness probe that checks DB connectivity (e.g.
"can I reach Mongo?"). If Mongo is temporarily unreachable, the webapp gets
restarted — but restarting the webapp doesn't fix Mongo. Use readiness here:
shed the traffic, don't kill the process.

### 4. Readiness probe path that doesn't exist → permanent NotReady

If you typo the path (`/heatlh` instead of `/health`), the pod will never
become Ready. Deployments will stall mid-rollout because the new pod never
joins the endpoints.

### 5. `timeoutSeconds` shorter than the probe actually needs

An `exec` probe runs inside the container via the kubelet. If the mongosh
command itself takes 3s and `timeoutSeconds: 1`, every probe call times out
→ probe fails → restart loop. Always measure the actual command duration first.

---

## Debugging exercise

### Scenario: liveness probe with wrong HTTP path → constant restarts

1. Edit `lessons/05-health-probes/webapp-deployment.yaml`. In the
   `livenessProbe` section, change:
   ```yaml
   path: /
   ```
   to:
   ```yaml
   path: /healthz
   ```
   The demo webapp has no `/healthz` endpoint — it will return 404, which
   is a probe failure.

2. Apply:
   ```bash
   kubectl apply -f lessons/05-health-probes/webapp-deployment.yaml
   ```

3. Watch:
   ```bash
   kubectl get pods --watch
   ```
   After about 30 seconds (3 failures × 10s period) you'll see `RESTARTS`
   increment on the webapp pod.

4. Diagnose:
   ```bash
   # Step 1 — check events on the pod
   kubectl describe pod -l app=webapp
   # Look for: Warning  Unhealthy  Liveness probe failed: HTTP probe failed
   #           Warning  Killing    ... Container webapp failed liveness probe

   # Step 2 — confirm the path is reachable manually
   kubectl exec -it <webapp-pod-name> -- wget -qO- http://localhost:3000/healthz
   # Expected: 404 Not Found — confirms the path doesn't exist
   ```

5. Fix: revert `path` back to `/` and re-apply. Confirm `RESTARTS` stops
   incrementing.

---

## Review questions

1. Explain the difference between a liveness probe and a readiness probe.
   What action does Kubernetes take when each one fails?

2. You have a Spring Boot app that takes 45–90 seconds to start (loading
   application context). What probe configuration would you use, and why
   is a `startupProbe` better than a large `initialDelaySeconds` on the
   liveness probe?

3. A webapp connects to a database at startup. Should you fail the readiness
   probe or the liveness probe if the database is unreachable? Justify your
   answer.

4. What is the total time Kubernetes will wait before restarting a container
   whose liveness probe has `periodSeconds: 15` and `failureThreshold: 4`?

5. Your CI/CD pipeline deploys a new version of the webapp. During rollout,
   the new pod gets 0% traffic for 20 seconds even though the container is
   Running. What probe is responsible for this behaviour, and is it working
   correctly?
