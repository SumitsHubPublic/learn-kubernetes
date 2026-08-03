# Lesson 03 — Kustomize Commands

All commands run from the repo root (`/home/sumit/learnings/k8s-intro`).

---

## Preview rendered manifests (no cluster changes)

```bash
# Render the dev overlay — check namespace=dev, replicas=1, no resources block
kubectl kustomize lessons/03-kustomize/overlays/dev

# Render the prod overlay — check namespace=prod, replicas=2, resources block on webapp
kubectl kustomize lessons/03-kustomize/overlays/prod
```

---

## Setup

```bash
# Create the namespaces Kustomize will deploy into
kubectl create namespace dev
kubectl create namespace prod
```

---

## Apply dev

```bash
# Apply all resources in the dev overlay (ConfigMap, Secret, Deployments, Services)
kubectl apply -k lessons/03-kustomize/overlays/dev
```

Expected output — each resource shows `created`:
```
configmap/mongo-config created
secret/mongo-secret created
deployment.apps/mongo-deployment created
service/mongo-service created
deployment.apps/webapp-deployment created
service/webapp-service created
```

---

## Verify dev

```bash
# List all resources in the dev namespace
kubectl get all -n dev

# Wait for all pods to be Running
kubectl wait --for=condition=Ready pods --all -n dev --timeout=60s

# Get the NodePort URL and open it in the browser
minikube service webapp-service -n dev --url
```

---

## Apply prod

```bash
# Apply all resources in the prod overlay (namespace=prod, webapp replicas=2, resource limits)
kubectl apply -k lessons/03-kustomize/overlays/prod
```

---

## Verify prod

```bash
# List all resources in the prod namespace — expect 2 webapp pods
kubectl get all -n prod

# Confirm resource limits are set on the webapp containers
kubectl describe pod -n prod -l app=webapp | grep -A6 "Limits:"

# Watch the webapp rollout reach 2/2 Ready
kubectl rollout status deployment/webapp-deployment -n prod
```

---

## Debugging exercise

```bash
# 1. Break the patch — introduce a typo in the container name
#    Edit overlays/prod/webapp-deployment-patch.yaml:
#    change  name: webapp
#    to      name: web-app

# 2. Render to see the problem before applying
kubectl kustomize lessons/03-kustomize/overlays/prod
# Look for TWO container entries in the webapp-deployment output

# 3. Apply the broken config
kubectl apply -k lessons/03-kustomize/overlays/prod

# 4. Watch pods fail
kubectl get pods -n prod -w

# 5. Describe to see the error
kubectl describe pod -n prod -l app=webapp
# Events will show: "container web-app: image not specified" or ErrImagePull

# 6. Revert the typo (name: webapp), then fix
kubectl apply -k lessons/03-kustomize/overlays/prod
kubectl rollout status deployment/webapp-deployment -n prod
```

---

## Diff an overlay (see what would change before applying)

```bash
# Show what kubectl apply -k would change vs the live cluster state
kubectl diff -k lessons/03-kustomize/overlays/prod
```

---

## Teardown (when done with this lesson)

```bash
# Delete all resources in both namespaces
kubectl delete -k lessons/03-kustomize/overlays/dev
kubectl delete -k lessons/03-kustomize/overlays/prod

# Optionally delete the namespaces themselves
kubectl delete namespace dev
kubectl delete namespace prod
```
