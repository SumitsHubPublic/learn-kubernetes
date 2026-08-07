# Lesson 9 — RBAC & Security commands

**RBAC** = Role-Based Access Control

Commands in order. Run from the repo root unless stated otherwise.

---

## Pre-requisite: Minikube with a NetworkPolicy-enforcing CNI

The default Minikube CNI does **not** enforce NetworkPolicy. To get enforcement,
re-create the cluster with Cilium:

```bash
minikube delete
minikube start --cni=cilium
```

Verify Cilium pods are running:
```bash
kubectl get pods -n kube-system -l k8s-app=cilium
# NAME             READY   STATUS    RESTARTS   AGE
# cilium-xxxxx     1/1     Running   0          60s
```

> Skip this if you only want to practice the securityContext and ServiceAccount
> parts — those work on any Minikube setup. NetworkPolicy rules apply to the
> cluster regardless, but won't be enforced without a supporting CNI.

---

## 1. (Optional) Label the namespace for Pod Security Admission

```bash
kubectl label namespace default \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

`enforce=baseline` rejects clearly dangerous pods.
`warn=restricted` shows warnings for anything that doesn't meet the restricted standard.

Verify the labels were set:
```bash
kubectl get namespace default --show-labels
```

---

## 2. Apply the ServiceAccounts

```bash
kubectl apply -f lessons/09-rbac-security/mongo-serviceaccount.yaml
kubectl apply -f lessons/09-rbac-security/webapp-serviceaccount.yaml
```

Verify they exist:
```bash
kubectl get serviceaccounts
# NAME        SECRETS   AGE
# default     0         ...
# mongo-sa    0         5s
# webapp-sa   0         5s
```

---

## 3. Apply the NetworkPolicies

```bash
kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml
kubectl apply -f lessons/09-rbac-security/webapp-network-policy.yaml
```

Verify they were created:
```bash
kubectl get networkpolicies
# NAME                    POD-SELECTOR   AGE
# mongo-network-policy    app=mongo      5s
# webapp-network-policy   app=webapp     5s
```

Inspect a policy in detail:
```bash
kubectl describe networkpolicy mongo-network-policy
```

---

## 4. Apply the base stack (if cluster is fresh)

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/07-persistent-volumes/mongo-pvc.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
kubectl apply -f lessons/04-ingress/webapp-service.yaml
```

---

## 5. Apply the updated Deployments

```bash
kubectl apply -f lessons/09-rbac-security/mongo-deployment.yaml
kubectl apply -f lessons/09-rbac-security/webapp-deployment.yaml
```

Watch the rolling update complete:
```bash
kubectl get pods --watch
# Both pods should reach 1/1 Running
```

---

## 6. Verify pods are running as non-root

```bash
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- id
# Expected: uid=999(mongodb) gid=999(mongodb) groups=999(mongodb)

kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- id
# Expected: uid=1000 gid=1000
```

---

## 7. Verify no API token is mounted

```bash
kubectl exec -it $(kubectl get pod -l app=mongo -o jsonpath='{.items[0].metadata.name}') \
  -- ls /var/run/secrets/kubernetes.io/serviceaccount/ 2>&1
# Expected: No such file or directory
```

---

## 8. Verify security context is applied on the pod

```bash
kubectl get pod -l app=mongo -o jsonpath='{.items[0].spec.securityContext}' | python3 -m json.tool
# Should show runAsNonRoot, runAsUser, fsGroup, seccompProfile
```

---

## 9. Verify NetworkPolicy enforcement (requires Cilium CNI)

Test that the webapp can reach Mongo:
```bash
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- nc -zv mongo-service 27017
# Expected: Connection to mongo-service 27017 port [tcp/*] succeeded!
```

Test that a test pod NOT labelled `app: webapp` cannot reach Mongo:
```bash
# Spin up a temporary pod without the webapp label
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600

# Try to connect to Mongo from the test pod — should be blocked
kubectl exec -it test-pod -- nc -zv mongo-service 27017 -w 3
# Expected: nc: mongo-service (10.x.x.x:27017): Connection timed out

# Clean up the test pod
kubectl delete pod test-pod
```

---

## 10. Inspect all security-related fields on a running pod

```bash
kubectl get pod -l app=mongo -o yaml | grep -A 20 "securityContext"
```

---

## 11. Check PSA violations (warn mode)

If you labelled the namespace with `warn=restricted`, any `kubectl apply` that
creates pods will print warnings for restricted violations. Example output:

```
Warning: would violate PodSecurity "restricted:latest": ...
```

Review and fix each warning before switching to `enforce=restricted`.

---

## 12. Debugging exercise — NetworkPolicy blocks webapp → Mongo

```bash
# 1. Break: edit mongo-network-policy.yaml — change podSelector app: webapp to app: wrong-label
kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml

# 2. Test the connection — should now time out
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- nc -zv mongo-service 27017 -w 3
# Expected: Connection timed out

# 3. Check webapp logs for errors
kubectl logs -l app=webapp

# 4. Diagnose the policy
kubectl describe networkpolicy mongo-network-policy
# Look at the "Allowing ingress traffic" section — podSelector should show wrong-label

# 5. Fix: revert podSelector to app: webapp
kubectl apply -f lessons/09-rbac-security/mongo-network-policy.yaml

# 6. Verify connection restored
kubectl exec -it $(kubectl get pod -l app=webapp -o jsonpath='{.items[0].metadata.name}') \
  -- nc -zv mongo-service 27017
# Expected: succeeded!
```

---

## 13. Clean up lesson 9 resources

```bash
kubectl delete -f lessons/09-rbac-security/webapp-deployment.yaml
kubectl delete -f lessons/09-rbac-security/mongo-deployment.yaml
kubectl delete -f lessons/09-rbac-security/webapp-network-policy.yaml
kubectl delete -f lessons/09-rbac-security/mongo-network-policy.yaml
kubectl delete -f lessons/09-rbac-security/webapp-serviceaccount.yaml
kubectl delete -f lessons/09-rbac-security/mongo-serviceaccount.yaml
```
