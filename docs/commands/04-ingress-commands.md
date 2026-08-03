# Lesson 4 — Ingress commands

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Enable the Minikube Ingress addon

```bash
minikube addons enable ingress
```
Installs the nginx Ingress Controller into the `ingress-nginx` namespace.

---

## 2. Wait for the controller pod to be Running

```bash
kubectl get pods -n ingress-nginx --watch
```
Wait until `ingress-nginx-controller-*` shows `1/1 Running`.
Press Ctrl+C once it's ready.

```bash
# One-shot check (no watch)
kubectl get pods -n ingress-nginx
```

---

## 3. Apply the base stack (if starting from a fresh cluster)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
kubectl apply -f lessons/01-k8s-basics/webapp.yaml
```
The last command creates the NodePort service from lesson 1.

---

## 4. Apply lesson 4 manifests

```bash
# Swap NodePort → ClusterIP on the webapp service
kubectl apply -f lessons/04-ingress/webapp-service.yaml

# Create the Ingress routing rule
kubectl apply -f lessons/04-ingress/webapp-ingress.yaml
```

---

## 5. Confirm the service type changed

```bash
kubectl get svc webapp-service
```
`TYPE` column should now show `ClusterIP` (not `NodePort`).

---

## 6. Check the Ingress got an address

```bash
kubectl get ingress webapp-ingress
```
Expected:
```
NAME             CLASS   HOSTS          ADDRESS        PORTS   AGE
webapp-ingress   nginx   webapp.local   192.168.49.2   80      30s
```
If `ADDRESS` is empty, the controller isn't ready yet — wait and retry.

---

## 7. Inspect Ingress details

```bash
kubectl describe ingress webapp-ingress
```
Shows the full rule set, backend resolution, and any events (errors appear
here first if something is misconfigured).

---

## 8. Get Minikube's IP

```bash
minikube ip
# typical output: 192.168.49.2
```

---

## 9. Add the hostname to /etc/hosts (WSL2)

```bash
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts
```
Run once. Verify with:
```bash
grep webapp.local /etc/hosts
```

---

## 10. Test the Ingress

```bash
# Full hostname request
curl http://webapp.local

# Explicit Host header (useful if /etc/hosts isn't set)
curl -H "Host: webapp.local" http://$(minikube ip)
```
Expected: webapp HTML page.

---

## 11. Confirm ClusterIP service is no longer reachable externally

```bash
# This should time out or refuse — the service is no longer NodePort
curl http://$(minikube ip):30007
```

---

## 12. Check service endpoints (used in debugging exercise)

```bash
kubectl get endpoints webapp-service
```
Shows which pod IPs and ports the service is pointing at.
Expected: something like `172.17.0.x:3000`.

---

## 13. Check Ingress Controller logs (advanced debugging)

```bash
kubectl logs -n ingress-nginx \
  $(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller \
    -o jsonpath='{.items[0].metadata.name}')
```
Look for `upstream` errors (502s) or `cannot find service` (wrong name/port).

---

## 14. Clean up lesson 4 resources

```bash
kubectl delete -f lessons/04-ingress/webapp-ingress.yaml
kubectl delete -f lessons/04-ingress/webapp-service.yaml
kubectl delete -f lessons/01-k8s-basics/webapp.yaml
kubectl delete -f lessons/01-k8s-basics/mongo.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-config.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-secret.yaml
```

Disable the addon (optional):
```bash
minikube addons disable ingress
```
