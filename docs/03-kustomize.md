# Lesson 03 — Kustomize

## What is Kustomize?

Kustomize is a **template-free** Kubernetes configuration manager built into
`kubectl` (since v1.14). Instead of templating YAML with `{{ }}` placeholders,
you write plain, valid Kubernetes YAML and then layer _transformations_ on top
of it. Every intermediate file is itself valid YAML you can `kubectl apply`
directly — nothing is hidden inside a rendering engine.

The core idea: one **base** (shared, environment-agnostic manifests) + one or
more **overlays** (per-environment differences). Kustomize merges them when you
run `kubectl kustomize` or `kubectl apply -k`.

---

## Kustomize vs Helm — when to choose which

| | Kustomize | Helm |
|---|---|---|
| Config format | Plain YAML + patch files | Go templates (`{{ }}`) |
| Package ecosystem | None (BYOK: bring your own kustomization) | Artifact Hub — thousands of charts |
| Diff/audit | Easy — every file is readable YAML | Harder — need `helm template` to see rendered output |
| Parameterisation | Patches, generators, transformers | Full `values.yaml` with conditionals and loops |
| Best for | Your own apps across envs (dev/prod) | Installing third-party software (Prometheus, cert-manager…) |
| Bundled in `kubectl` | Yes | No (separate install) |

**Rule of thumb:** Kustomize for _your_ workloads; Helm for _third-party_ charts.
Many production teams use both together (e.g. Helm chart for Postgres + Kustomize
overlays for their own services).

---

## Core concepts

### `kustomization.yaml`
The only required file. Kustomize looks for this name in every directory.
It declares what resources belong here and what transformations to apply.

### Base
A directory containing raw Kubernetes manifests plus a `kustomization.yaml`
that lists them under `resources:`. The base has **no environment-specific
knowledge** — it describes the application in its simplest valid form.

### Overlay
A directory with its own `kustomization.yaml` whose `resources:` points at
the base (or another overlay). The overlay adds environment-specific changes
— a different namespace, extra replicas, resource limits, a different image
tag, etc. — without touching the base files.

### Strategic Merge Patch
The most common way to customise a field. You write a **partial** YAML
document that looks like the original resource but only contains the fields
you want to change. Kustomize merges it on top of the base using Kubernetes'
own merge logic (e.g. containers are matched by `name`, not array index).

```yaml
# overlays/prod/webapp-deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment     # must match the base resource exactly
spec:
  replicas: 2                 # only this + resources below are changed
  template:
    spec:
      containers:
      - name: webapp          # matched by name, not position
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

Because Kubernetes strategic-merge logic matches list items by a merge key
(`name` for containers), you do **not** need to repeat every other container
field — only the fields you intend to change.

---

## What changed in this lesson

### New folder structure

```
lessons/03-kustomize/
  base/
    kustomization.yaml          ← lists all 6 resources
    mongo-config.yaml
    mongo-secret.yaml
    mongo-deployment.yaml
    mongo-service.yaml
    webapp-deployment.yaml
    webapp-service.yaml
  overlays/
    dev/
      kustomization.yaml        ← namespace=dev, points at base
    prod/
      kustomization.yaml        ← namespace=prod, applies patch
      webapp-deployment-patch.yaml  ← replicas=2 + resource limits
```

The base manifests are the same application as `lessons/01-k8s-basics/` but:
- Split into one file per resource (easier to patch individual resources).
- No inline comments (comments in base files survive patching but clutter diffs;
  put educational comments in the overlay or docs instead).

### dev overlay (`overlays/dev/kustomization.yaml`)
Only one change from base: every resource gets `namespace: dev` injected.
Kustomize adds this to every resource's `metadata.namespace` automatically.

### prod overlay (`overlays/prod/kustomization.yaml` + patch)
Two changes from base:
1. `namespace: prod` injected into every resource.
2. Strategic merge patch applies `replicas: 2` and CPU/memory
   `resources.requests` and `resources.limits` to the webapp container.
   MongoDB is left at 1 replica (stateful services need more care before scaling).

---

## How to apply

> **Note:** Kustomize is bundled in `kubectl`. No separate install needed.
> The `kustomize` standalone binary has more features but `kubectl kustomize`
> covers everything in this lesson.

### 1. Preview rendered output (dry run — no cluster changes)

```bash
kubectl kustomize lessons/03-kustomize/overlays/dev
kubectl kustomize lessons/03-kustomize/overlays/prod
```

Read through the output. Confirm:
- Every resource has `namespace: dev` (or `prod`).
- The prod `webapp-deployment` has `replicas: 2` and a `resources:` block.
- The dev `webapp-deployment` has `replicas: 1` and **no** `resources:` block.

### 2. Create the namespaces

```bash
kubectl create namespace dev
kubectl create namespace prod
```

### 3. Apply dev

```bash
kubectl apply -k lessons/03-kustomize/overlays/dev
```

### 4. Verify dev

```bash
kubectl get all -n dev
```

All pods should reach `Running`. Access the app:

```bash
minikube service webapp-service -n dev --url
```

Open the printed URL in your browser.

### 5. Apply prod

```bash
kubectl apply -k lessons/03-kustomize/overlays/prod
```

### 6. Verify prod

```bash
kubectl get all -n prod
```

You should see **2** webapp pods. Check resource limits are set:

```bash
kubectl describe pod -n prod -l app=webapp | grep -A6 "Limits:"
```

---

## Debugging exercise

### Scenario: patch silently misses its target

In a strategic merge patch, if the container `name` in the patch does not
exactly match the container `name` in the base, Kustomize **adds a new
container** instead of merging into the existing one — resulting in two
containers in the pod (one with resources set, one without), or sometimes
a pod that fails to start.

**To reproduce:**

1. Open `lessons/03-kustomize/overlays/prod/webapp-deployment-patch.yaml`.
2. Change the container name from `webapp` to `web-app` (add a hyphen).
3. Run: `kubectl kustomize lessons/03-kustomize/overlays/prod`
4. Look at the rendered `webapp-deployment` — you will see **two** container
   entries: the original `webapp` (no resources) and a new `web-app` (with
   resources but no image or ports).
5. Try applying: `kubectl apply -k lessons/03-kustomize/overlays/prod`
6. The pod will enter `CrashLoopBackOff` or `Error` because the injected
   container has no image.

**Diagnose:**

```bash
kubectl get pods -n prod
kubectl describe pod -n prod -l app=webapp
# Look for "container web-app: image not specified" in Events
```

**Fix:** revert the container name back to `webapp`.

```bash
kubectl apply -k lessons/03-kustomize/overlays/prod
kubectl rollout status deployment/webapp-deployment -n prod
```

**Key takeaway:** strategic merge patches match containers by `name`. Always
double-check the container name in your patch matches the base exactly —
especially when copying patches between projects.

---

## Common mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Wrong relative path in overlay `resources:` | `error: accumulating resources: ...no such file or directory` | Check that `../../base` resolves to the base dir |
| Container name typo in patch | Extra container added; pod crashloops | Match `name` exactly to base |
| Forgetting `namespace: <ns>` in overlay | Resources land in `default` namespace | Add `namespace:` field to overlay `kustomization.yaml` |
| Patching a field that doesn't exist yet | Field silently created with only patch's value | That's fine — unless you expected a merge |
| Using `patchesStrategicMerge:` (old key) | Deprecation warning in newer kubectl | Rename to `patches:` with `path:` entries |

---

## Other useful Kustomize features (not in this lesson)

### `namePrefix` / `nameSuffix`
Adds a prefix/suffix to every resource name. Common pattern:
```yaml
namePrefix: prod-
```
All resources become `prod-mongo-deployment`, `prod-webapp-service`, etc.
Useful when deploying the same app multiple times in one namespace.

### `commonLabels` / `commonAnnotations`
Injects a label/annotation into every resource and into selector fields:
```yaml
commonLabels:
  env: prod
  managed-by: kustomize
```

### `configMapGenerator` / `secretGenerator`
Generates a ConfigMap or Secret from a file or a list of `key=value` pairs,
and appends a content hash to the name:
```yaml
configMapGenerator:
- name: mongo-config
  literals:
  - mongo-url=mongo-service
```
The generated name becomes `mongo-config-<hash>`. This forces a rolling
restart of Deployments when the config changes — something plain ConfigMaps
don't do.

### `images`
Override an image name/tag without editing the base:
```yaml
images:
- name: nanajanashia/k8s-demo-app
  newTag: v2.0
```
This is the Kustomize equivalent of `helm upgrade --set image.tag=v2.0`.

---

## Review questions

1. What is the difference between a **base** and an **overlay** in Kustomize?
2. You apply the dev overlay and the webapp pods go to the `default` namespace
   instead of `dev`. What did you most likely forget?
3. A strategic merge patch is supposed to set `replicas: 3` on a Deployment,
   but after applying, `kubectl get deployment` still shows `replicas: 1`.
   Name two possible causes.
4. You need to deploy the same app twice in the same namespace (blue and green).
   Which Kustomize feature lets you do this without duplicating manifests?
5. Kustomize's `configMapGenerator` appends a hash to the ConfigMap name.
   Why is this considered _better_ than a static ConfigMap name in production?
