# Lesson 2 — Helm: command reference

Every command from lesson 2, in the order they were run, with what each one
does and what to expect. Copy-paste friendly — theory lives in
[`docs/02-helm.md`](../02-helm.md).

## Installation

```bash
# macOS with Homebrew
brew install helm
```

```bash
# Linux with apt (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install helm
```

```bash
# Linux with Curl (works on any Linux distro)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

```bash
# Windows with Chocolatey
choco install kubernetes-helm
```

```bash
# Windows with Scoop
scoop install helm
```

```bash
# Verify Helm is installed correctly
helm version
```
Expected: `version.BuildInfo{Version:"v3.x.x", ...}` showing your installed
Helm version.

```bash
# Verify kubectl is also available (Helm needs it to talk to the cluster)
kubectl version
```
Expected: both client and server versions should be available.

## Setup / scaffolding (already done for you)

```bash
# Scaffolds a new chart skeleton (Chart.yaml, values.yaml, templates/, a starter
# _helpers.tpl). We then deleted the generated nginx-example templates and
# wrote our own for mongo + webapp. Run from lessons/02-helm/.
helm create webapp-stack
```

## Validate the chart before touching the cluster

```bash
# Static checks: valid Chart.yaml, templates parse, values referenced exist, etc.
# Exits non-zero if something's structurally wrong - safe to run anytime, no cluster needed.
helm lint ./lessons/02-helm/webapp-stack
```
Expected output: `1 chart(s) linted, 0 chart(s) failed` (an `[INFO] icon is
recommended` line is harmless, ignore it).

```bash
# Renders every template to plain YAML using values.yaml, WITHOUT installing
# anything. This is your "preview" before applying to a real cluster - always
# run this after editing a template if you're not sure it's still valid.
helm template webapp ./lessons/02-helm/webapp-stack
```
Expected output: plain Kubernetes YAML for all 6 resources (Secret,
ConfigMap, 2 Services, 2 Deployments), separated by `---`, with no
`{{ }}` placeholders left - if you see literal `{{ }}` in the output, a
template has a typo.

## Move from raw manifests to a Helm-managed release

```bash
# Remove the resources that were created by `kubectl apply -f` on the lesson 1
# manifests earlier, so Helm can create its own copies without "already exists"
# name conflicts.
kubectl delete -f lessons/01-k8s-basics/
```

```bash
# Installs the chart as a new release named "webapp" into the current
# namespace (default). This is revision 1.
helm install webapp ./lessons/02-helm/webapp-stack
```
Expected output: `STATUS: deployed`, `REVISION: 1`.

```bash
# Prints the current status/metadata of the release (name, namespace,
# revision, status) - a quick sanity check, not the pod status.
helm status webapp
```

```bash
# Standard kubectl check - same as with raw manifests, Helm doesn't change how
# you inspect running resources, only how you install/upgrade/rollback them.
kubectl get all
```
Expected: both `mongo-deployment` and `webapp-deployment` pods `1/1 Running`.

```bash
# Confirms the app actually responds over the NodePort Service.
curl -s -m 5 http://$(minikube ip):30007
```
Expected: the app's HTML page (a "User profile" page in this tutorial image).

## Upgrade / history / rollback (the core Helm workflow - run these yourself)

```bash
# Re-renders the chart with an overridden value and applies the diff to the
# live release. Creates a NEW revision (2) - nothing is deleted/recreated from
# scratch, Kubernetes just reconciles the Deployment to the new spec.
helm upgrade webapp ./lessons/02-helm/webapp-stack --set webapp.replicaCount=2
```

```bash
kubectl get deploy webapp-deployment
```
Expected: `READY` column shows `2/2`.

```bash
# Lists every revision of this release: number, status, and the one-line
# description Helm auto-generates (or your --set/-f summary).
helm history webapp
```
Expected: two rows - revision 1 (`Install complete`) and revision 2
(`Upgrade complete`).

```bash
# Reverts the release to exactly how it looked at revision 1 - this creates a
# NEW revision (3) that happens to match revision 1's content; it does not
# delete revision 2 from history.
helm rollback webapp 1
```

```bash
kubectl get deploy webapp-deployment
helm history webapp
```
Expected: `READY` back to `1/1`, and `helm history` now shows 3 revisions
total (install, upgrade, rollback).

## Debugging exercise commands

```bash
# Deliberately deploys a nonexistent image tag to simulate a bad release.
helm upgrade webapp ./lessons/02-helm/webapp-stack --set webapp.image.tag=does-not-exist
```

```bash
kubectl get pods
```
Expected: a new `webapp-deployment-...` pod stuck in `ErrImagePull` or
`ImagePullBackOff`; the OLD webapp pod is still `Running` alongside it.

```bash
# Replace <pod-name> with the stuck pod's name from the previous command.
kubectl describe pod <pod-name>
```
Look at the `Events:` section at the very bottom - it names the exact image
and error (e.g. `manifest for nanajanashia/k8s-demo-app:does-not-exist not
found`).

```bash
# The app should still work - the old pod is still serving traffic.
curl -s -m 5 http://$(minikube ip):30007
```

```bash
kubectl get replicaset
```
Expected: two ReplicaSets for webapp - the old one at `1` desired/current
(still serving), the new broken one at `1` desired but `0` ready.

```bash
# Recover using history instead of guessing the right tag.
helm rollback webapp 1
kubectl get pods
kubectl get replicaset
```
Expected: pod list back to a single healthy `webapp-deployment` pod; the
broken ReplicaSet from the failed upgrade is scaled to 0 (still visible in
`kubectl get replicaset`, just with `0 0 0` - Helm/Kubernetes don't delete old
ReplicaSets automatically).

## Identifying Helm-managed vs raw manifest resources

### Check if a resource is managed by Helm

```bash
# List all active Helm releases - if your resource appears here, it's Helm-managed
helm list
```
Expected: `webapp` in the `NAME` column means resources created by our Helm chart.

```bash
# Check labels on the resource to see if it has Helm metadata
kubectl get deployment webapp-deployment --show-labels
```
Look for `app.kubernetes.io/managed-by=Helm` in the output.

```bash
# Full YAML view with all annotations and labels
kubectl get deployment webapp-deployment -o yaml
```
Helm-managed resources will have:
- Label: `app.kubernetes.io/managed-by: Helm`
- Annotations:
  - `meta.helm.sh/release-name: webapp`
  - `meta.helm.sh/release-namespace: default`

### View Helm's internal release storage

```bash
# Helm stores release metadata as Secrets in the cluster
kubectl get secrets -n default | grep -i helm
```
Expected: `sh.helm.release.v1.webapp.v1` (and similar for each revision).

```bash
# Try to get the release history - succeeds only if it's Helm-managed
helm history webapp
```
Expected: list of all revisions (install, upgrades, rollbacks).

### Compare Helm-managed vs raw kubectl

**With `kubectl apply -f` (raw manifests):**
- No `app.kubernetes.io/managed-by: Helm` label
- No `meta.helm.sh/release-*` annotations
- Resource won't appear in `helm list`
- No Helm release Secret in `kubectl get secrets`
- `helm history` command fails for that resource

**With Helm-managed resources:**
- All the metadata labels and annotations present
- Appears in `helm list`
- Release metadata stored as a Secret
- Full history available via `helm history`

## Other useful commands (not required this lesson, good to know)

```bash
helm get values webapp        # shows the values currently in effect for the release
helm get manifest webapp      # shows the exact rendered YAML currently applied
helm uninstall webapp         # removes the release and all its resources
```
