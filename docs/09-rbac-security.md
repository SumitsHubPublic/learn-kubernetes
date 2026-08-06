# Lesson 9 — RBAC & Security (PSA, NetworkPolicy, non-root)

**RBAC** = Role-Based Access Control

## What problem does this solve?

By the end of lesson 8 the stack is functionally correct: secrets are encrypted,
data is persistent, pods have health probes and resource limits. But every pod
is running as **root**, every pod has a Kubernetes API token mounted by default,
and there are no network rules preventing any pod from talking to any other pod.

This lesson hardens the runtime posture:

1. **Non-root containers** — pods run as unprivileged users.
2. **Security contexts** — capabilities are dropped, privilege escalation is
   blocked, and the seccomp profile is set.
3. **ServiceAccounts** — each workload gets a dedicated ServiceAccount with the
   API token explicitly disabled (neither Mongo nor the webapp need it).
4. **NetworkPolicy** — Mongo only accepts traffic from webapp; webapp only
   connects to Mongo.
5. **Pod Security Admission (PSA)** — a namespace-level control that rejects
   pods that don't meet a security standard.

---

## The four security layers

```
┌───────────────────────────────────────────────────────────────────┐
│ Layer 1 — Pod Security Admission (namespace label)                │
│   Kubernetes admission controller: reject pods that violate the   │
│   baseline or restricted standard BEFORE they are scheduled.      │
├───────────────────────────────────────────────────────────────────┤
│ Layer 2 — SecurityContext (per-pod / per-container)               │
│   What the container is ALLOWED to do once running:               │
│   runAsNonRoot, readOnlyRootFilesystem, capabilities, seccomp.    │
├───────────────────────────────────────────────────────────────────┤
│ Layer 3 — ServiceAccount + RBAC                                   │
│   What Kubernetes API resources the pod is ALLOWED to access.     │
│   Default SA auto-mounts a token. Dedicated SAs with no token     │
│   prevent pod code from calling the API at all.                   │
├───────────────────────────────────────────────────────────────────┤
│ Layer 4 — NetworkPolicy                                           │
│   What pods the container is ALLOWED to talk to over the network. │
│   Default: all pods talk to all pods. NetworkPolicy enforces       │
│   an explicit allow-list per pod selector.                         │
└───────────────────────────────────────────────────────────────────┘
```

Each layer is independent. Applying all four gives defence in depth: a
compromise that bypasses one layer is stopped by the next.

---

## Layer 1 — Pod Security Admission (PSA)

PSA is built into Kubernetes 1.25+ (GA). It replaces the deprecated
PodSecurityPolicy. You label a **namespace** and Kubernetes rejects any pod
creation that violates the chosen standard.

### Three standards

| Standard | What it allows |
|---|---|
| `privileged` | No restrictions — allows everything including privileged containers. |
| `baseline` | Blocks clearly dangerous settings: privileged containers, hostPID/hostNetwork, most capabilities. Still allows running as root. |
| `restricted` | Enforces current hardening best practices: non-root, no privilege escalation, seccomp required, all capabilities dropped. |

### Three modes

| Mode | Effect |
|---|---|
| `enforce` | Reject the pod if it violates the standard. |
| `warn` | Allow the pod but emit a warning in `kubectl` output. |
| `audit` | Allow the pod and record a violation in audit logs. |

You can mix modes for a gradual rollout — e.g. `warn` first so you see what
would be rejected, then switch to `enforce` once manifests are fixed.

### Applying PSA to a namespace

```bash
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

The `default` namespace now:
- **Enforces** `baseline`: pods that use privileged containers, hostPID,
  hostNetwork, or most capabilities are rejected.
- **Warns** about `restricted` violations (non-root, seccomp, etc.).
- **Audits** restricted violations to the audit log.

> **Note:** Minikube's `kube-system` namespace already has `privileged` enforce
> (because system pods like CoreDNS and the CNI need elevated permissions).
> Never apply `restricted` to `kube-system`.

---

## Layer 2 — SecurityContext

A `securityContext` can appear at two levels:

- **Pod-level** (`spec.securityContext`): applies to all containers in the pod.
- **Container-level** (`spec.containers[*].securityContext`): overrides pod-level
  for that specific container.

### Key fields

| Field | Where | What it does |
|---|---|---|
| `runAsNonRoot: true` | Pod | Reject container startup if the image's UID is 0 (root). |
| `runAsUser: N` | Pod | Force the container process to run as UID N. |
| `runAsGroup: N` | Pod | Force the container process to run as GID N. |
| `fsGroup: N` | Pod | Mounted volumes are chown'd to GID N so the process can write them. |
| `seccompProfile.type: RuntimeDefault` | Pod | Apply the container runtime's default seccomp filter (blocks ~300 dangerous syscalls). Required by the `restricted` PSA standard. |
| `allowPrivilegeEscalation: false` | Container | Prevent `setuid` binaries or `sudo` from gaining root. |
| `readOnlyRootFilesystem: true` | Container | Container can only write to explicitly mounted volumes and `emptyDir`. |
| `capabilities.drop: [ALL]` | Container | Drop all Linux capabilities (NET_RAW, SYS_ADMIN, etc.). Start from zero. |
| `capabilities.add: [NET_BIND_SERVICE]` | Container | Add back only what's needed (e.g. binding to ports < 1024). |

### MongoDB-specific notes

The official `mongo:8` image:
- Runs the `mongod` process as UID `999` (the `mongodb` OS user inside the image).
- Requires write access to `/data/db` (PVC mount) and `/tmp`.
- Cannot use `readOnlyRootFilesystem: true` because it writes to internal paths
  at startup.

The demo webapp image:
- Runs Node.js as UID `1000`.
- Does not write to the filesystem — `readOnlyRootFilesystem: true` is safe.

---

## Layer 3 — ServiceAccounts

Every pod in Kubernetes runs under a ServiceAccount (default: `default`).
By default, Kubernetes **auto-mounts a token** for that ServiceAccount into the
pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Any code running
in the container can use that token to call the Kubernetes API.

Neither Mongo nor the webapp needs to call the Kubernetes API. Mounting a token
they don't use is an unnecessary attack surface.

### What we change

1. Create a dedicated ServiceAccount per workload (`mongo-sa`, `webapp-sa`).
2. Set `automountServiceAccountToken: false` on each SA.
3. Reference the SA in the Deployment spec.

This means even if an attacker gains RCE inside the container, they have no
Kubernetes API credentials to work with.

### RBAC — when you DO need API access

If a workload needs to call the Kubernetes API (e.g. a controller, an operator,
or a CI/CD agent inside the cluster), you grant it precisely-scoped permissions
via RBAC:

```
ServiceAccount  ←  bound by  →  RoleBinding  ←  references  →  Role
                                                              (list of rules)
```

- **Role** — namespaced; grants permissions within one namespace.
- **ClusterRole** — cluster-scoped; grants permissions across all namespaces.
- **RoleBinding** — binds a Role (or ClusterRole) to a ServiceAccount in a namespace.
- **ClusterRoleBinding** — binds a ClusterRole cluster-wide.

The principle of least privilege: grant only the specific verbs (`get`, `list`,
`watch`, `create`, etc.) on the specific resource kinds needed.

Our workloads need no API access at all, so we don't create any Roles or
Bindings — just ServiceAccounts with the token disabled.

---

## Layer 4 — NetworkPolicy

By default, all pods in a Kubernetes cluster can communicate with all other pods
across all namespaces. There are no firewall rules between them.

NetworkPolicy is a Kubernetes resource that acts as a pod-level firewall:

```
Without NetworkPolicy:          With NetworkPolicy:
webapp ──► mongo ✓              webapp ──► mongo ✓
webapp ──► any-pod ✓            webapp ──► any-pod ✗
any-pod ──► mongo ✓             any-pod ──► mongo ✗
mongo ──► internet ✓            mongo ──► internet ✗
```

### Important: NetworkPolicy requires a CNI plugin that supports it

The cluster's CNI (Container Network Interface) plugin must enforce NetworkPolicy.
**Minikube's default CNI does not enforce NetworkPolicy.** You must enable the
`network-policy` addon or switch to a CNI that supports it (Calico, Cilium, etc.).

Enable it on Minikube:
```bash
# Option 1: use the cilium CNI (recommended for Minikube)
minikube start --cni=cilium

# Option 2: if already running, you can't hot-swap the CNI —
# re-create the cluster with cilium
minikube delete && minikube start --cni=cilium
```

> **Note:** the NetworkPolicy YAMLs apply successfully to any cluster regardless
> of CNI support, but they are silently ignored if the CNI doesn't enforce them.
> Always verify enforcement (see commands doc).

### Our NetworkPolicy design

**Mongo NetworkPolicy** (`mongo-network-policy.yaml`):
- Ingress: only allow port 27017 from pods with `app: webapp`.
- Egress: only allow DNS (UDP/TCP 53) — Mongo doesn't initiate outbound connections.

**Webapp NetworkPolicy** (`webapp-network-policy.yaml`):
- Ingress: allow port 3000 from anywhere in the cluster (Ingress controller,
  other services, health checks).
- Egress: only allow port 27017 to pods with `app: mongo`, plus DNS.

---

## What changes in this lesson

### New files

| File | Purpose |
|---|---|
| `lessons/09-rbac-security/mongo-serviceaccount.yaml` | SA for Mongo, token disabled |
| `lessons/09-rbac-security/webapp-serviceaccount.yaml` | SA for webapp, token disabled |
| `lessons/09-rbac-security/mongo-network-policy.yaml` | Isolate Mongo to webapp-only ingress |
| `lessons/09-rbac-security/webapp-network-policy.yaml` | Isolate webapp egress to Mongo only |
| `lessons/09-rbac-security/mongo-deployment.yaml` | Mongo deployment with securityContext + SA |
| `lessons/09-rbac-security/webapp-deployment.yaml` | Webapp deployment with securityContext + SA |

### What was added to each deployment

**Both deployments:**
- `spec.serviceAccountName` — references the dedicated SA.
- `spec.securityContext.runAsNonRoot: true` — pod-level non-root enforcement.
- `spec.securityContext.seccompProfile.type: RuntimeDefault` — meets PSA restricted/baseline.
- `spec.containers[*].securityContext.allowPrivilegeEscalation: false` — blocks setuid/sudo.
- `spec.containers[*].securityContext.capabilities.drop: [ALL]` — drops all Linux capabilities.

**Webapp only:**
- `spec.securityContext.runAsUser: 1000` — runs as Node.js user.
- `spec.containers[*].securityContext.readOnlyRootFilesystem: true` — immutable filesystem.

**Mongo only:**
- `spec.securityContext.runAsUser: 999` + `fsGroup: 999` — MongoDB user + PVC write access.
- `readOnlyRootFilesystem: false` — MongoDB writes internally; cannot be fully read-only.

---

## How to apply

### Step 1 — (Optional) Label the namespace for PSA

```bash
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

In `warn` mode you'll see warnings in the output of subsequent `kubectl apply`
commands — this is useful to see what still needs fixing before enforcing.

### Step 2 — Apply ServiceAccounts

```bash
kubectl apply -f lessons/09-rbac-security/mongo-serviceaccount.yaml
kubectl apply -f lessons/09-rbac-security/webapp-serviceaccount.yaml
```

### Step 3 — Apply NetworkPolicies

```bash
kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml
kubectl apply -f lessons/09-rbac-security/webapp-network-policy.yaml
```

### Step 4 — Apply updated Deployments

```bash
kubectl apply -f lessons/09-rbac-security/mongo-deployment.yaml
kubectl apply -f lessons/09-rbac-security/webapp-deployment.yaml
```

Pods are replaced immediately via a rolling update.

### Step 5 — Verify pods are running as non-root

```bash
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- id
# uid=999(mongodb) gid=999(mongodb)

kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- id
# uid=1000 gid=1000
```

### Step 6 — Verify no API token is mounted

```bash
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1
# ls: cannot access '/var/run/secrets/kubernetes.io/serviceaccount/': No such file or directory
```

---

## Common mistakes

### 1. Pod fails to start with `container has runAsNonRoot and image will run as root`

This means the image's default user is root (UID 0) and you've set
`runAsNonRoot: true` without specifying a `runAsUser`. Either:
- Set `runAsUser: <non-zero-uid>` to override the image's default.
- Rebuild the image with a `USER` directive in the Dockerfile.

### 2. PVC is not writable after adding `fsGroup`

If you add `fsGroup` but the volume already exists with different ownership,
Kubernetes will recursively `chown` the volume on pod startup. For large
databases this can take a long time. Use `fsGroupChangePolicy: OnRootMismatch`
to only chown when the root directory's GID doesn't already match.

### 3. NetworkPolicy blocks DNS → pods can't resolve hostnames

Always include DNS egress rules (UDP/TCP port 53) in your NetworkPolicy.
Without them, pods with an egress NetworkPolicy can't resolve service names
like `mongo-service` and all connections fail with `NXDOMAIN`.

### 4. NetworkPolicy appears to do nothing

The CNI plugin must support NetworkPolicy enforcement. On Minikube, the default
CNI does NOT enforce NetworkPolicy. You must start Minikube with
`--cni=cilium` (or another supporting CNI). Check: `kubectl get pods -n kube-system`
should show `cilium-*` pods if Cilium is active.

### 5. PSA rejects existing pods in `kube-system`

Never label `kube-system` with `restricted` or `baseline` enforce. System
components (CoreDNS, kube-proxy, CNI pods) legitimately need elevated
permissions. Only label your own application namespaces.

### 6. `readOnlyRootFilesystem: true` breaks the app

If the app writes temp files, logs, or caches to arbitrary paths inside the
container (not to a mounted volume), it will crash with `Read-only file system`.
Fix by:
- Mounting an `emptyDir` volume at the path(s) the app writes to.
- Or setting `readOnlyRootFilesystem: false` and documenting why.

---

## Debugging exercise

### Scenario: NetworkPolicy blocks webapp → Mongo → application returns database errors

This simulates accidentally locking Mongo too tightly so even the webapp
can't reach it.

**Break it:**

1. Edit `lessons/09-rbac-security/mongo-network-policy.yaml`. In the `ingress`
   section, change the `podSelector` to use a label that doesn't match the
   webapp — for example `app: wrong-label`.

2. Apply the broken policy:
   ```bash
   kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml
   ```

3. Open the webapp in the browser (via `minikube service webapp-service --url`
   or through the Ingress). You should see a database connection error.

**Diagnose it:**

```bash
# Check the webapp pod logs for connection errors
kubectl logs -l app=webapp

# Confirm the NetworkPolicy is applied and inspect the podSelector
kubectl describe networkpolicy mongo-network-policy

# Test connectivity manually from webapp pod to mongo service
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- nc -zv mongo-service 27017
# Should fail (connection refused or timeout)
```

**Fix it:**

Revert the `podSelector` to `app: mongo` and re-apply:

```bash
kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml
```

Test again — the connection should succeed.

---

## Review questions

1. What is the difference between `enforce`, `warn`, and `audit` modes in Pod
   Security Admission? When would you use `warn` first instead of going straight
   to `enforce`?

2. Why does the Mongo deployment set `fsGroup: 999` but the webapp deployment
   does not? What does `fsGroup` actually control?

3. You add `readOnlyRootFilesystem: true` to a container and it crashes on
   startup. What is the most likely cause, and what are two ways to fix it?

4. Every NetworkPolicy with an egress rule must include a DNS egress rule.
   Why? What happens if you omit it?

5. `automountServiceAccountToken: false` is set on both ServiceAccounts. If a
   future pod needs to `kubectl get pods` from inside the cluster, what RBAC
   objects would you create to allow it with minimum privilege?
