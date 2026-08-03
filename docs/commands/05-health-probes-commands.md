# Lesson 5 — Health Probes commands

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Apply the base stack (if cluster is fresh)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml   # creates mongo Service
```

---

## 2. Apply lesson 5 deployments (with probes)

```bash
kubectl apply -f lessons/05-health-probes/mongo-deployment.yaml
kubectl apply -f lessons/05-health-probes/webapp-deployment.yaml
```

---

## 3. Watch pods come up

```bash
kubectl get pods --watch
```
Expect `READY 0/1` while startup/readiness probes are pending, then `1/1` once
they pass. Press Ctrl+C when both pods are `1/1 Running`.

---

## 4. Confirm probe config on the webapp pod

```bash
kubectl describe pod -l app=webapp | grep -A 20 "Liveness\|Readiness"
```
Shows the httpGet path, port, thresholds, and delay settings.

---

## 5. Confirm probe config on the mongo pod

```bash
kubectl describe pod -l app=mongo | grep -A 20 "Startup\|Liveness\|Readiness"
```
Shows the exec command, thresholds, and startup probe budget.

---

## 6. Watch probe events in real time

```bash
kubectl get events --sort-by='.lastTimestamp' --watch
```
`Unhealthy` events appear when any probe fails. Press Ctrl+C to stop.

---

## 7. Check pod restart count

```bash
kubectl get pods
```
`RESTARTS` column shows how many times a container has been restarted by a
failed liveness or startup probe.

---

## 8. View logs of the previous (restarted) container

```bash
# Useful after a liveness probe restart loop
kubectl logs -l app=webapp --previous
kubectl logs -l app=mongo  --previous
```
`--previous` shows the logs from the last terminated instance of the container.

---

## 9. Manually test the webapp health endpoint from inside the pod

```bash
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- http://localhost:3000/
```
A successful response (HTML page) means the liveness/readiness endpoint works.

---

## 10. Manually run the Mongo ping from inside the pod

```bash
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh --eval "db.adminCommand('ping')" --quiet
```
Expected output: `{ ok: 1 }`

---

## 11. Simulate probe failure — wrong liveness path (debugging exercise)

```bash
# 1. Edit webapp-deployment.yaml: change livenessProbe.httpGet.path from / to /healthz
# 2. Apply
kubectl apply -f lessons/05-health-probes/webapp-deployment.yaml

# 3. Watch restarts accumulate
kubectl get pods --watch

# 4. See the Unhealthy event
kubectl describe pod -l app=webapp
# Look for: Warning  Unhealthy  Liveness probe failed: HTTP probe failed with
#           statuscode: 404

# 5. Confirm the path doesn't exist
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- http://localhost:3000/healthz

# 6. Fix: revert path back to / and re-apply
kubectl apply -f lessons/05-health-probes/webapp-deployment.yaml
```

---

## 12. Check endpoints (readiness gate for traffic)

```bash
kubectl get endpoints webapp-service
```
When the webapp readiness probe is failing, the pod IP disappears from this
list — no traffic is routed to it. When it passes again, the IP reappears.

---

## 13. Clean up lesson 5 resources

```bash
kubectl delete -f lessons/05-health-probes/webapp-deployment.yaml
kubectl delete -f lessons/05-health-probes/mongo-deployment.yaml
kubectl delete -f lessons/01-k8s-basics/mongo.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-config.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-secret.yaml
```
