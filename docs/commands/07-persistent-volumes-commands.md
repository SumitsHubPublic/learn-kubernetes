# Lesson 7 — Persistent Volumes commands

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Apply the base stack (if cluster is fresh)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
kubectl apply -f lessons/01-k8s-basics/webapp.yaml
```

---

## 2. Create the PVC

```bash
kubectl apply -f lessons/07-persistent-volumes/mongo-pvc.yaml
```

---

## 3. Check PVC status

```bash
kubectl get pvc mongo-data-pvc
```
`STATUS: Bound` = dynamically provisioned and ready. `Pending` = problem (see debugging exercise).

---

## 4. Inspect PVC details and events

```bash
kubectl describe pvc mongo-data-pvc
```
Shows the provisioned PV name, capacity, access mode, and any error events.

---

## 5. List all PersistentVolumes in the cluster

```bash
kubectl get pv
```
Shows the PV that was auto-created for your PVC, its reclaim policy, and status.

---

## 6. Check available StorageClasses

```bash
kubectl get storageclass
```
Minikube ships with one StorageClass named `standard` (provisioner: `k8s.io/minikube-hostpath`).

---

## 7. Apply the Mongo deployment with volumeMount

```bash
kubectl apply -f lessons/07-persistent-volumes/mongo-deployment.yaml
```

---

## 8. Confirm the volume is mounted in the pod

```bash
kubectl describe pod -l app=mongo | grep -A 5 "Mounts\|Volumes"
```
Expected: `/data/db from mongo-data (rw)` and `ClaimName: mongo-data-pvc`.

---

## 9. Decode your Mongo credentials (needed for mongosh)

```bash
kubectl get secret mongo-secret -o jsonpath='{.data.mongo-root-username}' | base64 -d && echo
kubectl get secret mongo-secret -o jsonpath='{.data.mongo-root-password}' | base64 -d && echo
```

---

## 10. Open a mongosh session inside the Mongo pod

```bash
kubectl exec -it \
  $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u <username> -p <password> --authenticationDatabase admin
```

---

## 11. Insert a test document (inside mongosh)

```js
use testdb
db.lesson7.insertOne({ message: "data survives pod restart", ts: new Date() })
db.lesson7.find()
exit
```

---

## 12. Delete the pod to prove persistence

```bash
# Delete the pod — Deployment will recreate it immediately
kubectl delete pod -l app=mongo

# Watch the new pod come up
kubectl get pods --watch
```

---

## 13. Query the new pod to confirm data survived

```bash
kubectl exec -it \
  $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u <username> -p <password> --authenticationDatabase admin
```
```js
use testdb
db.lesson7.find()
# Should return the document inserted before the pod was deleted
exit
```

---

## 14. Simulate PVC Pending — debugging exercise

```bash
# 1. Edit mongo-pvc.yaml: change storageClassName to "fast"
# 2. Delete and re-create the PVC
kubectl delete -f lessons/07-persistent-volumes/mongo-pvc.yaml
kubectl apply  -f lessons/07-persistent-volumes/mongo-pvc.yaml

# 3. Check status
kubectl get pvc mongo-data-pvc
# STATUS: Pending

# 4. Diagnose
kubectl describe pvc mongo-data-pvc
# Look for: Events — no persistent volumes available for this claim

# 5. Check what StorageClasses actually exist
kubectl get storageclass

# 6. Fix: revert storageClassName to "standard", then:
kubectl delete -f lessons/07-persistent-volumes/mongo-pvc.yaml
kubectl apply  -f lessons/07-persistent-volumes/mongo-pvc.yaml
kubectl get pvc mongo-data-pvc
# STATUS: Bound
```

---

## 15. Clean up lesson 7 resources

```bash
kubectl delete -f lessons/07-persistent-volumes/mongo-deployment.yaml
kubectl delete -f lessons/07-persistent-volumes/mongo-pvc.yaml    # WARNING: deletes the PV and all data (Minikube default)
kubectl delete -f lessons/01-k8s-basics/webapp.yaml
kubectl delete -f lessons/01-k8s-basics/mongo.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-config.yaml
kubectl delete -f lessons/01-k8s-basics/mongo-secret.yaml
```
