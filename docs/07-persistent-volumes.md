# Lesson 7 — Persistent Volumes

## What problem does this solve?

Right now Mongo stores all its data inside the container's **ephemeral
filesystem**. That layer lives and dies with the container process. Every
time the pod restarts — because of an OOM kill (lesson 6), a node reboot,
a rolling update, or even just a `kubectl delete pod` — all database records
are gone.

For a real application this is catastrophic. Lesson 7 fixes it by attaching
a **PersistentVolumeClaim** to the Mongo pod, so the data directory lives
outside the container on a volume that outlives any individual pod.

---

## The three-layer storage model

Kubernetes separates storage into three objects:

```
┌─────────────────────────────────────────────────────────────────┐
│  StorageClass  ←  describes HOW to provision storage            │
│  e.g. "standard" (Minikube hostPath), "gp2" (AWS EBS), etc.    │
│                                                                 │
│  PersistentVolume (PV)  ←  a piece of actual storage           │
│  Could be a directory on a node, an NFS mount, a cloud disk.   │
│  With dynamic provisioning the StorageClass creates it for you. │
│                                                                 │
│  PersistentVolumeClaim (PVC)  ←  your request for storage      │
│  "I need 1 GiB, ReadWriteOnce."                                 │
│  Kubernetes binds a matching PV to it.                          │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │  pod references the PVC by name
   mongo-deployment (volumeMount → /data/db)
```

As an application developer you almost never create PVs directly. You:
1. Write a **PVC** declaring what you need (size, access mode, storage class).
2. Mount the PVC into your pod with a `volumeMount`.
3. The **StorageClass** automatically provisions the backing PV for you.

---

## Static vs dynamic provisioning

| | Static | Dynamic |
|---|---|---|
| Who creates the PV? | A cluster admin, manually | The StorageClass provisioner, automatically when a PVC is created |
| When to use | On-prem clusters with pre-allocated storage arrays | Cloud providers and Minikube — the normal path |
| Our setup | Dynamic | Minikube's `standard` class provisions a hostPath volume on the node |

---

## Access modes

| Mode | Short | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | One node can mount read/write. Typical for databases. |
| `ReadOnlyMany` | ROX | Many nodes can mount read-only. |
| `ReadWriteMany` | RWX | Many nodes can mount read/write. Requires NFS or a distributed filesystem. |

> **Important:** RWO means one *node*, not one *pod*. Multiple pods on the
> same node can technically mount an RWO volume, but databases like MongoDB
> cannot safely share a data directory between two running instances.
> Always run a single Mongo pod when using an RWO PVC.

---

## Reclaim policies

When you delete a PVC, what happens to the backing PV and its data?

| Policy | Behaviour |
|---|---|
| `Delete` | PV and the underlying storage are deleted automatically. Default for dynamically provisioned volumes. |
| `Retain` | PV stays around (status: `Released`); data is preserved for manual recovery. Good for production — never auto-delete DB data. |
| `Recycle` (deprecated) | PV is wiped and made available again. |

For production databases you should set `reclaimPolicy: Retain` on the
StorageClass or override it on the PV. Minikube's `standard` class defaults
to `Delete` — fine for dev, not for production.

---

## What changed in this lesson

### `lessons/07-persistent-volumes/mongo-pvc.yaml` — new PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-data-pvc
spec:
  storageClassName: standard
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

This requests 1 GiB of storage from Minikube's `standard` provisioner.
Kubernetes dynamically creates a PV backed by a directory on the Minikube
node and binds it to this claim.

### `lessons/07-persistent-volumes/mongo-deployment.yaml` — volumeMount added

Two additions to the existing Deployment (resource limits and probes from
lessons 5 and 6 are carried forward):

```yaml
spec:
  template:
    spec:
      containers:
      - name: mongodb
        volumeMounts:
        - name: mongo-data
          mountPath: /data/db    # ← must be exactly this; it's where Mongo writes its files
      volumes:
      - name: mongo-data
        persistentVolumeClaim:
          claimName: mongo-data-pvc
```

`volumes` declares what volumes exist at the pod level.  
`volumeMounts` wires a named volume into a specific path inside the container.  
The `name` field (`mongo-data`) is just a label linking the two — it only has
to be consistent within the same pod spec.

`/data/db` is MongoDB's hardcoded default data directory. If you mount to any
other path you must also pass `--dbpath` to the mongod process. Don't change it.

---

## How to apply

### Step 1 — Apply the base stack

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml       # mongo Service (unchanged)
kubectl apply -f lessons/01-k8s-basics/webapp.yaml      # webapp Deployment + Service
```

### Step 2 — Create the PVC

```bash
kubectl apply -f lessons/07-persistent-volumes/mongo-pvc.yaml
```

Check it was bound:
```bash
kubectl get pvc mongo-data-pvc
```
Expected:
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mongo-data-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi        RWO            standard       5s
```
`STATUS: Bound` means a PV was dynamically provisioned and attached. If it
stays `Pending`, see the debugging exercise.

### Step 3 — Apply the Mongo deployment with the volume

```bash
kubectl apply -f lessons/07-persistent-volumes/mongo-deployment.yaml
```

### Step 4 — Confirm the volume is mounted in the pod

```bash
kubectl describe pod -l app=mongo | grep -A 5 "Mounts\|Volumes"
```

Expected output (partial):
```
Mounts:
  /data/db from mongo-data (rw)
Volumes:
  mongo-data:
    Type:       PersistentVolumeClaim
    ClaimName:  mongo-data-pvc
```

### Step 5 — Prove persistence survives a pod restart

This is the payoff of the lesson. Do this step carefully.

```bash
# 1. Open a shell inside the Mongo pod
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u <your-username> -p <your-password> --authenticationDatabase admin

# 2. Inside mongosh, insert a test document
use testdb
db.lesson7.insertOne({ message: "data survives pod restart", ts: new Date() })
db.lesson7.find()
# You should see the document
exit

# 3. Delete the pod — the Deployment will immediately recreate it
kubectl delete pod -l app=mongo

# 4. Wait for the new pod to be 1/1 Running
kubectl get pods --watch

# 5. Open a new shell into the NEW pod and query again
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- mongosh -u <your-username> -p <your-password> --authenticationDatabase admin

use testdb
db.lesson7.find()
# The document is still there — the PVC persisted it
exit
```

Replace `<your-username>` and `<your-password>` with the values from your
`mongo-secret.yaml` (decode them: `echo "<base64>" | base64 -d`).

---

## StatefulSets — the production path for databases

In this lesson we use a **Deployment** with a manually created PVC. This is
correct for learning but has one limitation: if you scale to `replicas: 2`,
both Mongo pods share the same PVC and the same data directory — MongoDB will
corrupt the data.

In production, Mongo (and any stateful app) should run as a **StatefulSet**.
A StatefulSet uses `volumeClaimTemplates` to automatically create one PVC per
replica pod with a stable identity (`mongo-0`, `mongo-1`, etc.). This is the
foundation for running a proper Mongo replica set.

We cover StatefulSets implicitly in later lessons, but the important takeaway
now is: **keep `replicas: 1` on this Deployment** until we properly configure
a replica set.

---

## Common mistakes

### 1. PVC stays Pending forever

Cause: the `storageClassName` doesn't match any available StorageClass in the
cluster, or the requested `storage` exceeds what the provisioner can allocate.

Diagnose: `kubectl describe pvc mongo-data-pvc` — the Events section will say
something like `no persistent volumes available for this claim`.

Fix: `kubectl get storageclass` to see what's available in your cluster, then
match the name exactly.

### 2. Pod stuck in `ContainerCreating` after applying the deployment

If the PVC is still `Pending`, the pod will wait indefinitely in
`ContainerCreating`. Fix the PVC first — the pod will automatically proceed
once the PVC becomes `Bound`.

### 3. Changing `mountPath` away from `/data/db`

MongoDB writes all data to `/data/db` by default. Mounting to `/mongo-data`
or `/data` means Mongo starts a fresh empty instance at `/data/db` and ignores
your PVC. The data appears to be lost even though the PVC is healthy.

### 4. Deleting the PVC while the pod is running

Kubernetes will prevent this (PVC is protected by a finalizer while in use).
But if you delete the pod first and then the PVC, the dynamically provisioned
PV is also deleted (Minikube's `Delete` reclaim policy), and all data is gone.
In production, use `reclaimPolicy: Retain`.

### 5. Scaling Deployment replicas > 1 with a single RWO PVC

The second pod will be stuck in `ContainerCreating` because the RWO PVC is
already mounted on a node (only one node can hold it). The second pod will
only schedule if it lands on the same node — and if it does, MongoDB will
corrupt the data. Use StatefulSets for multi-replica databases.

---

## Debugging exercise

### Scenario: wrong storageClassName → PVC never Bound → pod stuck Pending

1. Edit `lessons/07-persistent-volumes/mongo-pvc.yaml`. Change:
   ```yaml
   storageClassName: standard
   ```
   to:
   ```yaml
   storageClassName: fast
   ```
   (`fast` doesn't exist in Minikube.)

2. Apply:
   ```bash
   kubectl apply -f lessons/07-persistent-volumes/mongo-pvc.yaml
   ```

3. Check the PVC status:
   ```bash
   kubectl get pvc mongo-data-pvc
   # STATUS stays: Pending
   ```

4. Diagnose:
   ```bash
   kubectl describe pvc mongo-data-pvc
   # Events section: no persistent volumes available for this claim and no storage
   # class is set  — or — waiting for a volume to be created

   # Confirm available StorageClasses
   kubectl get storageclass
   # Only "standard" is listed — "fast" doesn't exist
   ```

5. Fix: revert `storageClassName` back to `standard` and re-apply the PVC.
   It should transition to `Bound` within a few seconds.

---

## Review questions

1. Explain the relationship between a StorageClass, a PersistentVolume, and
   a PersistentVolumeClaim. Which one do you create as an application
   developer?

2. What is the difference between `ReadWriteOnce` and `ReadWriteMany`? Give
   an example of a workload that would need `ReadWriteMany`.

3. If you delete the Mongo pod (but not the PVC or Deployment), what happens
   to the data? What if you delete the PVC too (Minikube default reclaim policy)?

4. Why should you NOT scale the Mongo Deployment to `replicas: 2` with the
   current setup? What Kubernetes object would you use instead for a
   production multi-replica database?

5. A pod is stuck in `ContainerCreating` for 5 minutes after you applied its
   Deployment. What are the two most likely causes, and how would you
   diagnose them?
