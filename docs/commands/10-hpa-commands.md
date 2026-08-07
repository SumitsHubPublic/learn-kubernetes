# Lesson 10 — Horizontal Pod Autoscaler commands

**HPA** = Horizontal Pod Autoscaler

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Enable Metrics Server on Minikube

```bash
minikube addons enable metrics-server
```

Verify the pod comes up (can take ~30-60s):
```bash
kubectl get pods -n kube-system -l k8s-app=metrics-server
# NAME                              READY   STATUS    RESTARTS   AGE
# metrics-server-xxxxxxxxxx-xxxxx   1/1     Running   0          45s
```

Confirm the metrics API is actually serving data (may show an error for the
first ~1 minute — that's normal, wait and retry):
```bash
kubectl top nodes
kubectl top pods
```

---

## 2. Apply the base stack (if cluster is fresh)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/07-persistent-volumes/mongo-pvc.yaml
kubectl apply -f lessons/09-rbac-security/mongo-serviceaccount.yaml
kubectl apply -f lessons/09-rbac-security/webapp-serviceaccount.yaml
kubectl apply -f lessons/09-rbac-security/mongo-deployment.yaml
kubectl apply -f lessons/09-rbac-security/webapp-deployment.yaml
kubectl apply -f lessons/04-ingress/webapp-service.yaml
```

Confirm both pods are running before continuing:
```bash
kubectl get pods
```

---

## 3. Apply the HorizontalPodAutoscaler

```bash
kubectl apply -f lessons/10-hpa/webapp-hpa.yaml
```

Verify it exists and check the initial state:
```bash
kubectl get hpa webapp-hpa
# NAME         REFERENCE                    TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
# webapp-hpa   Deployment/webapp-deployment cpu: 1%/50%, mem: 20%/80%   2         5          2          20s
```

> If `TARGETS` shows `<unknown>/50%`, give it 30-60s for Metrics Server to
> collect its first data point. If it never resolves, see the debugging
> exercise below.

Inspect full detail, including recent scaling events:
```bash
kubectl describe hpa webapp-hpa
```

---

## 4. Watch the replica count settle at `minReplicas`

```bash
kubectl get deployment webapp-deployment --watch
# replicas should climb from 1 to 2 (minReplicas) shortly after the HPA is applied
```

Press `Ctrl+C` to stop watching once it settles.

---

## 5. Generate load to trigger a scale-up

```bash
kubectl apply -f lessons/10-hpa/load-generator.yaml
```

In a second terminal, watch the HPA react:
```bash
kubectl get hpa webapp-hpa --watch
```

In a third terminal, watch replicas actually being added:
```bash
kubectl get pods -l app=webapp --watch
```

Give it 1-3 minutes — you should see `TARGETS` cross 50%, then replica count
climb toward `maxReplicas: 5`.

---

## 6. Stop the load and watch it scale back down

```bash
kubectl delete -f lessons/10-hpa/load-generator.yaml
```

```bash
kubectl get hpa webapp-hpa --watch
```

Because `scaleDown.stabilizationWindowSeconds: 60` and at most 1 pod is
removed per 60s policy window, expect this to take a few minutes to settle
back to 2 replicas — it's deliberately cautious, not instant.

---

## 7. Inspect the full HPA object as YAML

```bash
kubectl get hpa webapp-hpa -o yaml
```

Look at `status.currentMetrics` vs `spec.metrics`, and `status.conditions`
for `AbleToScale` / `ScalingActive` / `ScalingLimited`.

---

## 8. Debugging exercise — `<unknown>` targets from a missing CPU request

```bash
# 1. Break: remove/comment the resources.requests.cpu line on the webapp
#    container in lessons/06-resource-limits/webapp-deployment.yaml (or your
#    live copy in lessons/09-rbac-security/webapp-deployment.yaml), then:
kubectl apply -f lessons/09-rbac-security/webapp-deployment.yaml

# 2. Observe the broken state
kubectl get hpa webapp-hpa
# TARGETS column shows <unknown>/50%, <unknown>/80%

# 3. Diagnose
kubectl describe hpa webapp-hpa
# Conditions show: "unable to get metrics for resource cpu: missing request for cpu"

kubectl get deployment webapp-deployment -o jsonpath='{.spec.template.spec.containers[0].resources}'
# confirms requests.cpu is absent

# 4. Fix: restore resources.requests.cpu: "50m" and re-apply
kubectl apply -f lessons/09-rbac-security/webapp-deployment.yaml

# 5. Verify it recovers within one sync interval (~15s)
kubectl get hpa webapp-hpa --watch
```

---

## 9. Clean up lesson 10 resources

```bash
kubectl delete -f lessons/10-hpa/load-generator.yaml --ignore-not-found
kubectl delete -f lessons/10-hpa/webapp-hpa.yaml
```
