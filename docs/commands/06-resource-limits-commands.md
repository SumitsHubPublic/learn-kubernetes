# Lesson 6 — Resource Requests & Limits commands

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Apply the base stack (if cluster is fresh)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
```

---

## 2. Apply lesson 6 deployments

```bash
kubectl apply -f lessons/06-resource-limits/mongo-deployment.yaml
kubectl apply -f lessons/06-resource-limits/webapp-deployment.yaml
```

---

## 3. Verify resource requests and limits on the webapp pod

```bash
kubectl describe pod -l app=webapp | grep -A 6 "Limits\|Requests"
```
Expected:
```
Limits:
  cpu:     200m
  memory:  128Mi
Requests:
  cpu:     50m
  memory:  64Mi
```

---

## 4. Verify resource requests and limits on the mongo pod

```bash
kubectl describe pod -l app=mongo | grep -A 6 "Limits\|Requests"
```
Expected:
```
Limits:
  cpu:     500m
  memory:  512Mi
Requests:
  cpu:     100m
  memory:  256Mi
```

---

## 5. Check QoS class of each pod

```bash
kubectl get pod -l app=webapp -o jsonpath='{.items[0].status.qosClass}{"\n"}'
kubectl get pod -l app=mongo  -o jsonpath='{.items[0].status.qosClass}{"\n"}'
```
Both should print `Burstable`.

---

## 6. View total resource allocation on the Minikube node

```bash
kubectl describe node minikube | grep -A 10 "Allocated resources"
```
Shows CPU and memory requests vs node capacity across all running pods.

---

## 7. View actual live resource usage (requires metrics-server)

```bash
# Check if metrics-server is running
kubectl get pods -n kube-system | grep metrics-server

# Enable it in Minikube if not present
minikube addons enable metrics-server

# Wait ~60s, then check usage
kubectl top pod
kubectl top node
```
`kubectl top pod` shows actual CPU/memory consumption vs requests.

---

## 8. Check the full resource spec of a running pod

```bash
kubectl get pod -l app=webapp -o jsonpath='{.items[0].spec.containers[0].resources}' \
  | python3 -m json.tool
```

---

## 9. Simulate OOMKill — debugging exercise

```bash
# 1. Edit webapp-deployment.yaml: set limits.memory to "10Mi"

# 2. Apply
kubectl apply -f lessons/06-resource-limits/webapp-deployment.yaml

# 3. Watch the crash loop
kubectl get pods --watch

# 4. Diagnose the OOMKill
kubectl describe pod -l app=webapp
# Look for: Last State: Terminated  Reason: OOMKilled  Exit Code: 137

# 5. Fix: revert memory limit to 128Mi and re-apply
kubectl apply -f lessons/06-resource-limits/webapp-deployment.yaml
```

---

## 10. Check restart reason and exit code directly

```bash
kubectl get pod -l app=webapp \
  -o jsonpath='{.items[0].status.containerStatuses[0].lastState.terminated}' \
  | python3 -m json.tool
```
OOMKilled shows `"reason": "OOMKilled"` and `"exitCode": 137`.

---

## 11. Clean up lesson 6 resources

```bash
kubectl delete -f lessons/06-resource-limits/webapp-deployment.yaml
kubectl delete -f lessons/06-resource-limits/mongo-deployment.yaml
kubectl delete -f lessons/01-k8s-basics/mongo.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-config.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-secret.yaml
```
