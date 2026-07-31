# Lesson 1 — K8s Basics: command reference

Every command relevant to this lesson, with what each one does and what to
expect. Copy-paste friendly — theory lives in
[`docs/01-k8s-basics.md`](../01-k8s-basics.md).

## Apply the manifests

```bash
# Applies every YAML file in the folder (all 4 manifests: 2 Deployments,
# 2 Services, 1 ConfigMap, 1 Secret spread across mongo.yaml/webapp.yaml).
kubectl apply -f lessons/01-k8s-basics/
```
Expected output: one line per created/unchanged object, e.g.
`deployment.apps/mongo-deployment created`, `service/mongo-service created`,
`configmap/mongo-config created`, `secret/mongo-secret created`, and the same
for webapp.

## Inspect what's running

```bash
# Everything at a glance: pods, services, deployments, replicasets.
kubectl get all
```

```bash
# Pods only, with node placement and IP - useful for "which node is this on".
kubectl get pods -o wide
```
Expected: `mongo-deployment-xxxxx` and `webapp-deployment-xxxxx`, both
`1/1 Running`.

```bash
# Full detail on one object: config, events, conditions - your first stop
# whenever something isn't behaving as expected.
kubectl describe pod <pod-name>
kubectl describe deployment webapp-deployment
```

```bash
# Container stdout/stderr - check this whenever a Pod is Running but the app
# seems broken (Kubernetes only tracks process-alive, not app-healthy).
kubectl logs deployment/webapp-deployment
kubectl logs deployment/mongo-deployment
```

## Confirm the ConfigMap/Secret values landed correctly

```bash
# Shows the ConfigMap's actual key/value data.
kubectl get configmap mongo-config -o yaml
```

```bash
# Shows the Secret - values are base64, NOT decrypted for you (by design of
# `kubectl get -o yaml`; decode manually if you need to check a value).
kubectl get secret mongo-secret -o yaml
```

```bash
# Manually decode one value to see the real password - demonstrates why
# base64 isn't real security (docs/01-k8s-basics.md section on Secrets).
kubectl get secret mongo-secret -o jsonpath='{.data.mongo-root-password}' | base64 -d
```

## Reach the app

```bash
# Get Minikube's node IP, then curl the NodePort Service.
minikube ip
curl -s -m 5 http://$(minikube ip):30007
```
Expected: the app's HTML page.

## Debugging exercise commands

```bash
# Overrides DB_URL directly, bypassing the ConfigMap - simulates a bad config change.
kubectl set env deployment/webapp-deployment DB_URL=wrong-service-name
```

```bash
kubectl get pods
kubectl logs deployment/webapp-deployment
```
Expected: Pod still shows `Running`; logs likely show a Mongo connection
error once the app tries to use the (now wrong) `DB_URL`.

```bash
curl -s -m 5 http://$(minikube ip):30007
```
Expected: page still loads (static HTML), but anything hitting the backend
API (e.g. `/get-profile`) would fail - open browser dev tools / curl that
endpoint directly to see the failure if you want to go further.

```bash
# Resets the Deployment's env block back to exactly what's in the file
# (including restoring the configMapKeyRef for DB_URL).
kubectl apply -f lessons/01-k8s-basics/webapp.yaml
```

```bash
kubectl get pods
curl -s -m 5 http://$(minikube ip):30007
```
Expected: back to fully working.

## Cleanup (only if you want to remove everything from lesson 1)

```bash
# Delete all resources defined in the lesson 1 folder
kubectl delete -f lessons/01-k8s-basics/
```

```bash
# Alternative: delete specific resource types
kubectl delete deployment mongo-deployment
kubectl delete deployment webapp-deployment
kubectl delete service mongo-service
kubectl delete service webapp-service
kubectl delete configmap mongo-config
kubectl delete secret mongo-secret
```

```bash
# Verify everything is cleaned up
kubectl get all
```
Expected: no resources related to mongo or webapp should remain.

```bash
# Nuclear option - delete ALL resources in the default namespace
# (use with extreme caution!)
kubectl delete all --all
```
