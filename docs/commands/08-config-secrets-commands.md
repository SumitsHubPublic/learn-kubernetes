# Lesson 8 — Config & Secrets Management commands

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Add the Sealed Secrets Helm repo

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
```

---

## 2. Install the Sealed Secrets controller

```bash
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system \
  -f lessons/08-config-secrets/sealed-secrets-values.yaml
```

---

## 3. Verify the controller pod is Running

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
```
Wait for `1/1 Running`.

---

## 4. Install the kubeseal CLI (Linux / WSL2)

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest \
  | grep '"tag_name"' | sed 's/.*"v\([^"]*\)".*/\1/')

curl -sSL \
  "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" \
  | tar xz kubeseal

sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

---

## 5. Fetch and inspect the controller's public key

```bash
kubeseal --fetch-cert \
  --controller-name sealed-secrets \
  --controller-namespace kube-system
```
This is the public key used to encrypt. Outputs PEM — safe to share/commit.

---

## 6. Seal the mongo secret

```bash
kubeseal \
  --format yaml \
  --namespace default \
  < lessons/01-k8s-basics/mongo-secret.yaml \
  > lessons/08-config-secrets/mongo-sealed-secret.yaml
```
The output file contains real RSA+AES-256-GCM ciphertext — safe to commit.

---

## 7. Inspect the generated SealedSecret

```bash
cat lessons/08-config-secrets/mongo-sealed-secret.yaml
```
Confirm `kind: SealedSecret` and that `encryptedData` values look like long
opaque strings (not recognisable base64).

---

## 8. Remove the plain Secret if it already exists

```bash
kubectl delete secret mongo-secret --ignore-not-found
```

---

## 9. Apply the SealedSecret

```bash
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml
```

---

## 10. Confirm the Secret was created by the controller

```bash
kubectl get secret mongo-secret
```
Expected:
```
NAME           TYPE     DATA   AGE
mongo-secret   Opaque   2      3s
```

---

## 11. Verify the decrypted values are correct

```bash
kubectl get secret mongo-secret \
  -o jsonpath='{.data.mongo-root-username}' | base64 -d && echo
kubectl get secret mongo-secret \
  -o jsonpath='{.data.mongo-root-password}' | base64 -d && echo
```

---

## 12. List all SealedSecrets in the cluster

```bash
kubectl get sealedsecrets -A
```

---

## 13. Describe a SealedSecret (shows events / errors)

```bash
kubectl describe sealedsecret mongo-secret -n default
```
If the Secret was not created, the Events section will show the decryption error.

---

## 14. Back up the controller's master key — do this now

```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml
```
**Store this file securely — NOT in git.** You need it to restore sealed
secrets if the cluster is recreated.

---

## 15. Re-seal after a credential rotation

```bash
# 1. Update lessons/01-k8s-basics/mongo-secret.yaml locally with new values
# 2. Re-run kubeseal
kubeseal \
  --format yaml \
  --namespace default \
  < lessons/01-k8s-basics/mongo-secret.yaml \
  > lessons/08-config-secrets/mongo-sealed-secret.yaml

# 3. Apply
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml
```

---

## 16. Debugging exercise — wrong namespace

```bash
# 1. Create a test namespace
kubectl create namespace staging

# 2. Apply sealed secret to wrong namespace
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml -n staging

# 3. Check — Secret should NOT exist in staging
kubectl get secret mongo-secret -n staging

# 4. Diagnose the SealedSecret status
kubectl describe sealedsecret mongo-secret -n staging
# Look for: error decrypting key

# 5. Clean up and fix
kubectl delete sealedsecret mongo-secret -n staging
kubectl delete namespace staging
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml -n default
```

---

## 17. Uninstall Sealed Secrets (cleanup)

```bash
helm uninstall sealed-secrets -n kube-system
# The plain Secrets created by the controller remain; only the controller is removed
```
