# Lesson 2 — Helm

Files live in `lessons/02-helm/` · Commands used: [`docs/commands/02-helm-commands.md`](commands/02-helm-commands.md)

> This lesson (and lesson 1) was committed directly to `main` as part of an
> initial repo restructuring, so you could see the whole course skeleton in
> place before starting the branch-per-lesson workflow. Starting at lesson 3
> (Kustomize), we go back to the normal flow: work happens on
> `lesson/NN-topic` and merges to `main` once verified — see
> [`docs/00-roadmap.md`](00-roadmap.md).

## 0. Installing Helm

Helm is a standalone binary that runs on your machine (not in the cluster).
You only need to install it once.

### macOS

```bash
# Using Homebrew (recommended)
brew install helm

# Or with Curl
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Linux

```bash
# Using package manager (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install helm

# Or with Curl (works on any Linux)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Windows

```powershell
# Using Chocolatey
choco install kubernetes-helm

# Or using Scoop
scoop install helm
```

### Verify installation

```bash
helm version
```

Expected output: `version.BuildInfo{Version:"v3.x.x", ...}` (exact version
number varies depending on when you install).

```bash
# You should also have kubectl working (lesson 1 prerequisite)
kubectl version
```

Both `helm` and `kubectl` commands should be available in your terminal.

## 1. What problem does Helm solve?

Up to now your whole setup was 4 hand-written YAML files
(`mongo.yaml`, `mongo-config.yaml`, `mongo-secret.yaml`, `webapp.yaml`) applied
with `kubectl apply -f`. That works, but think about what happens when you
need a second environment — say a `staging` namespace with 1 replica and a
`prod` one with 3 replicas and a different image tag:

- **Without Helm:** copy-paste the YAML files, hand-edit the numbers/tags in
  each copy, remember to keep them in sync forever. Nothing stops you from
  applying the wrong file to the wrong cluster. If you `kubectl apply` a bad
  change, there's no built-in "undo" — you have to remember what the previous
  YAML looked like (git helps, but you must actually go find and re-apply it).
- **With Helm:** you write the YAML *once*, as a **template** with
  placeholders (`{{ .Values.webapp.replicaCount }}` instead of a hardcoded
  `1`). Then you supply different small `values.yaml` files per environment.
  Every install/change is tracked as a numbered **revision**, and you can
  `helm rollback` to any previous revision with one command.

**Helm is essentially a package manager for Kubernetes** — the same idea as
`npm` for Node packages or `apt` for Ubuntu packages, but the "package" is a
bundle of Kubernetes manifests plus the values that fill them in.

## 2. Core Helm vocabulary (new terms, explained)

| Term | What it actually is |
|---|---|
| **Chart** | A directory (or `.tgz`) containing templated Kubernetes YAML + metadata. Your chart is `webapp-stack/`. Think "the package". |
| **Template** | A YAML file under `templates/` that isn't plain YAML — it has `{{ ... }}` placeholders filled in at install/upgrade time. Uses Go's `text/template` syntax. |
| **Values** | The input data for the placeholders. Defaults live in `values.yaml` inside the chart; you can override them with `--set key=value` on the CLI or a separate `-f my-values.yaml` file. |
| **Release** | One installed *instance* of a chart in a cluster, with a name (e.g. `webapp`). You could install the same chart twice under two different release names into two namespaces — e.g. `helm install webapp-staging` and `helm install webapp-prod`. |
| **Revision** | A numbered snapshot of a release's rendered manifests + values, created every time you `helm install`/`upgrade`/`rollback`. This is what makes rollback possible — Helm keeps history, `kubectl apply` does not. |
| **Rendering** | The process of turning a template + values into final plain YAML. `helm template` shows you this without touching the cluster — extremely useful for debugging *before* you install anything. |

## 3. Why/when to use Helm (and when not to)

**Use Helm when:**
- You need the same app deployed with variations (dev/staging/prod, or
  multiple customers) without copy-pasting YAML.
- You want a rollback story that doesn't rely on `git revert` + manual
  re-apply.
- You're installing third-party software (Prometheus, cert-manager, Argo CD —
  all the things coming up later in this course) — almost everything in the
  Kubernetes ecosystem ships as a Helm chart today.

**Trade-offs / when it's overkill:**
- For a single static environment that never changes, plain manifests (or
  Kustomize, lesson 2) are simpler to read — there's no templating syntax to
  learn, and `kubectl diff`/`apply` is more transparent.
- Go template syntax mixed into YAML is genuinely harder to read than plain
  YAML, especially for people new to Helm (you're living that right now).
  This is *the* most common complaint about Helm.
- Helm gives you enough rope to hang yourself: `--set` on the command line is
  easy to mistype (`--set webapp.replicaCont=2` silently creates a *new*,
  unused value instead of erroring — nothing validates the key exists unless
  you add JSON Schema validation, which is out of scope for this lesson).

## 4. Common production mistakes with Helm

1. **Committing real secrets into `values.yaml`.** Our `mongo.auth.password`
   in `values.yaml` is a plaintext value that gets base64-encoded at render
   time — fine for local learning, a real incident in production. Lesson 7
   fixes this properly.
2. **Manually `kubectl edit`-ing a Helm-managed resource.** The next
   `helm upgrade` will silently overwrite your manual edit (or, depending on
   the change, cause a confusing diff). If it's Helm-managed, change it via
   `values.yaml`, not `kubectl edit`/`patch`.
3. **Not pinning chart versions for third-party charts.** When you later
   `helm install` things like `ingress-nginx` or `prometheus`, always pin
   `--version x.y.z` — an unpinned chart install today can be a different
   (breaking) chart version tomorrow.
4. **Forgetting `--atomic`/`--wait` in CI.** By default `helm upgrade` returns
   as soon as the API server accepts the change — it does **not** wait for
   pods to actually become healthy. In automated pipelines (lesson 11) you
   want `helm upgrade --atomic` so a failed rollout auto-rolls-back instead of
   leaving a broken release "successfully" installed.
5. **Mixing raw `kubectl apply -f` and Helm for the same resources.** We did
   exactly this transition in this lesson (deleted the raw manifests, then
   `helm install`) — the danger is doing it *partially*, where some resources
   are Helm-managed and an old raw one is left behind with the same name,
   causing "resource already exists" errors on install.

## 5. What actually changed in this repo

New directory: `lessons/02-helm/webapp-stack/` — a Helm chart replacing the
lesson 1 raw manifests (`lessons/01-k8s-basics/*.yaml`) with a templated,
values-driven equivalent.

```
lessons/02-helm/webapp-stack/
├── Chart.yaml              # chart metadata: name, version, appVersion
├── values.yaml              # the "input data" — image tags, replica count, ports, mongo creds
└── templates/
    ├── _helpers.tpl          # reusable name/label snippets (from `helm create`, trimmed down)
    ├── mongo-configmap.yaml  # was mongo-config.yaml
    ├── mongo-secret.yaml     # was mongo-secret.yaml — values now come from values.yaml, base64-encoded at render time
    ├── mongo-deployment.yaml # was half of mongo.yaml
    ├── mongo-service.yaml    # was the other half of mongo.yaml
    ├── webapp-deployment.yaml# was half of webapp.yaml
    └── webapp-service.yaml   # was the other half of webapp.yaml
```

Key differences from the raw YAML, explained:

- **Hardcoded values became `{{ .Values.xxx }}` placeholders.** E.g.
  `replicas: 1` became `replicas: {{ .Values.webapp.replicaCount }}`, so
  changing replica count is a values change, not a YAML edit.
- **Image repo/tag are split and combined in the template**:
  `image: "{{ .Values.webapp.image.repository }}:{{ .Values.webapp.image.tag }}"`
  — lets you override just the tag (e.g. for a new release) without touching
  the repository name.
- **Labels come from `_helpers.tpl` via `include`.** `webapp-stack.labels`
  adds standard `app.kubernetes.io/*` labels (name, instance, version,
  managed-by) automatically, on every resource, so you don't hand-type them
  and don't forget one on a new resource.
- **The Secret's `mongo-root-username`/`password` are computed with `b64enc`**
  instead of you pre-computing base64 by hand like the original
  `mongo-secret.yaml` did (`bW9uZ29hZG1pbg==` was literally
  `echo -n mongoadmin | base64`, already stale/hard to change safely).
- **`webapp-service.yaml`'s `nodePort` is conditional** on
  `.Values.webapp.service.type == "NodePort"` — this anticipates lesson 3,
  where we'll switch this to `ClusterIP` + Ingress and the `nodePort` field
  would otherwise be invalid to even have present.

The 4 original raw manifest files now live under `lessons/01-k8s-basics/` (see
[`docs/01-k8s-basics.md`](01-k8s-basics.md)) — kept in the repo on purpose so
you can `diff` them against the rendered chart output (see commands doc) and
see the 1:1 mapping between "plain manifest" and "Helm template".

## 6. Verification (what "done" looks like)

Full step-by-step commands with expected output are in
[`docs/commands/01-helm-commands.md`](commands/01-helm-commands.md). Summary
of what we verified together:

- `helm lint` and `helm template` — chart renders without errors, and the
  rendered YAML matches the original manifests field-for-field.
- `helm install webapp ./webapp-stack` — both pods reached `Running`, and
  `curl` against the NodePort returned the app's HTML page.
- `helm upgrade webapp ./webapp-stack --set webapp.replicaCount=2` — you ran
  this, confirmed `kubectl get deploy webapp-deployment` showed 2/2, then
  used `helm rollback webapp 1` to go back to 1 replica and confirmed
  `helm history` shows both the upgrade and the rollback as separate
  revisions.

## 7. How to identify Helm-managed vs raw manifest resources

Over time, you might inherit a cluster or forget how you deployed something.
Here's how to tell whether a resource was created with Helm or raw `kubectl
apply -f` manifests:

### Quick check: use `helm list`

```bash
# List all Helm releases
helm list

# If your resource appears here, it's Helm-managed
# If nothing appears, it was likely deployed with kubectl
```

### Deep dive: check resource labels and annotations

Every resource created by Helm gets special metadata. Run:

```bash
# Check labels on a resource
kubectl get deployment webapp-deployment --show-labels

# Or get the full YAML
kubectl get deployment webapp-deployment -o yaml
```

**Helm-managed resources have:**
- **Label:** `app.kubernetes.io/managed-by: Helm`
- **Annotations like:**
  - `meta.helm.sh/release-name: webapp`
  - `meta.helm.sh/release-namespace: default`

Example from a Helm-managed resource:

```yaml
metadata:
  labels:
    app.kubernetes.io/instance: webapp       # release name
    app.kubernetes.io/managed-by: Helm       # ← Helm magic
    app.kubernetes.io/name: webapp-stack
    app.kubernetes.io/version: "1.0"
  annotations:
    meta.helm.sh/release-name: webapp        # ← Helm release info
    meta.helm.sh/release-namespace: default  # ← Helm namespace
```

**Raw manifests (kubectl apply -f) have:**
- No `app.kubernetes.io/managed-by: Helm` label
- No `meta.helm.sh/release-*` annotations
- Often just basic labels you hand-typed (if any)

### Check Helm's internal storage

Helm stores release metadata as Secrets. If they exist, the resources are
Helm-managed:

```bash
# List all Helm release secrets
kubectl get secrets -n default | grep -i helm

# Expected output for a release named 'webapp':
# sh.helm.release.v1.webapp.v1          Opaque   1      2d
```

### View Helm release history

```bash
# See all revisions of a release
helm history webapp

# If this command succeeds, it's Helm-managed
# If it fails ("release 'webapp' not found"), it was kubectl apply -f
```

### Practical example: comparing both

**Helm-managed resource:**
```bash
$ kubectl get deployment webapp-deployment -o yaml | grep -A5 "labels:"
  labels:
    app.kubernetes.io/instance: webapp
    app.kubernetes.io/managed-by: Helm
    ...
```

**Raw manifest resource:**
```bash
$ kubectl get deployment nginx -o yaml | grep -A5 "labels:"
  labels:
    app: nginx
```

The Helm one has structured `app.kubernetes.io/*` labels; the raw one just
has whatever you typed.

### Why this matters

If you try to edit a Helm-managed resource with `kubectl edit` or `patch`, the
next `helm upgrade` will silently overwrite your changes. Always check which
tool owns the resource first — if it's Helm, change it via `values.yaml` and
`helm upgrade`, not `kubectl edit`.

## 8. Debugging exercise

Run this against your `webapp` release — it's meant to simulate a real "why
did my deploy break" moment:

1. Intentionally deploy a bad image tag:
   ```bash
   helm upgrade webapp ./lessons/02-helm/webapp-stack --set webapp.image.tag=does-not-exist
   ```
2. Watch the pod fail:
   ```bash
   kubectl get pods
   ```
   You should see the new webapp pod stuck in `ErrImagePull` or
   `ImagePullBackOff` instead of `Running`.
3. Diagnose it like you would a real incident — don't just guess, confirm:
   ```bash
   kubectl describe pod <the-stuck-webapp-pod-name>
   ```
   Look in the `Events:` section at the bottom for the actual pull error
   message (this is where Kubernetes tells you *why*, not just *that*, it
   failed).
4. Notice that the **old pod is still running and still serving traffic** —
   this is the Deployment controller's default rolling update behavior: it
   won't kill the old ReplicaSet's pod until the new one is confirmed healthy.
   Confirm the app is still reachable on NodePort 30007 while the new pod is
   broken.
5. Recover using Helm's history instead of guessing the right tag:
   ```bash
   helm rollback webapp 1
   kubectl get pods
   ```
6. Optional deeper dig: run `kubectl get replicaset` before and after step 5.
   Notice Kubernetes doesn't delete old ReplicaSets on rollback — it scales
   the matching old one back up to replace the broken one. This is also *why*
   rollback is instant: the ReplicaSet object (and its pod template) already
   existed, Kubernetes just changes which one is scaled to N and which is 0.

**What this teaches:** in production, a bad deploy rarely means instant full
outage — Kubernetes' rolling update is a safety net — but it *does* mean
degraded capacity/no rollout progress until someone notices and reacts. This
is exactly why lesson 4 (health probes) matters: without a correct
`readinessProbe`, Kubernetes can be fooled into thinking a broken pod is fine.

## 9. Git commit

This lesson was committed straight to `main` together with lesson 1, as part
of the initial repo restructuring into `lessons/NN-topic/` folders:

```
feat(lesson02): convert manifests to a Helm chart

Replace the lesson 1 raw manifests with a Helm chart at
lessons/02-helm/webapp-stack/, templating replica count, image repo/tag,
service type/port, and mongo credentials via values.yaml. Verified with
helm lint/template, a live install on minikube, and an upgrade/rollback
cycle.
```

From lesson 3 onward we go back to the branch-per-lesson workflow: work on
`lesson/03-kustomize`, verify, then merge to `main` — see
[`docs/00-roadmap.md`](00-roadmap.md).

## 10. Review questions

1. In your own words: what's the difference between a Helm **chart**, a
   **release**, and a **revision**? (Hint: one is "the package", one is "an
   installed instance of it", one is "a numbered snapshot of that instance".)
2. Why did `kubectl apply -f webapp.yaml` (old workflow) *not* give you an
   equivalent to `helm rollback`? What would you have had to do manually to
   "roll back" with the old approach?
3. During the debugging exercise, why did the app stay reachable even though
   the new pod was stuck in `ImagePullBackOff`? What Kubernetes object made
   that possible?
4. What's the actual security exposure of the `mongo-secret.yaml` template
   using `b64enc`, compared to the original hand-typed base64 values? (Trick
   question — think about whether anything actually got more secure.)
5. Name one real situation (not from this lesson) where you'd choose *not*
   to use Helm, and explain why in terms of trade-offs from section 3.
