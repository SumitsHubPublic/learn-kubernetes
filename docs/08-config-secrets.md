# Lesson 8 — Config & Secrets Management (Sealed Secrets)

## What problem does this solve?

Open `lessons/01-k8s-basics/mongo-secret.yaml`:

```yaml
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

Those values look encrypted. They are not. Base64 is an **encoding**, not
encryption — anyone can reverse it in one command:

```bash
echo "cGFzc3dvcmQ=" | base64 -d
# password
```

If this file is committed to a git repository — even a private one — your
credentials are exposed to every person with repo access, every CI runner,
every third-party integration, and the entire git history forever.

The core tension in GitOps is: **you want everything in git, but secrets
cannot go into git as plaintext (or base64).** This lesson solves that.

---

## Two main approaches — Sealed Secrets vs SOPS

Both approaches let you commit encrypted secret files to git safely.
They differ in where the decryption key lives.

### Sealed Secrets (Bitnami)

```
Developer                    Cluster
──────────                   ───────
mongo-secret.yaml  ──kubeseal──►  encrypted SealedSecret YAML  ──git commit──► repo
                                                                        │
                                                        ┌───────────────▼──────────┐
                                                        │  Sealed Secrets controller│
                                                        │  (has the private key)    │
                                                        │  decrypts → Secret object │
                                                        └──────────────────────────┘
```

- The controller holds a **private key** in the cluster.
- You use the `kubeseal` CLI to encrypt a plain Secret with the controller's
  **public key**.
- Only that specific cluster can decrypt the resulting `SealedSecret`.
- The encrypted YAML is safe to commit to git.

**Pros:** simple workflow, tight kubectl integration, no external key store needed.  
**Cons:** sealed secrets are tied to one cluster's key. Rotate the cluster
(or lose the key) and you must re-seal everything. Back up the controller
key (`kubectl get secret -n kube-system sealed-secrets-key... -o yaml`).

### SOPS + age (Mozilla / Filippo Valsorda)

```
Developer                              Repository
──────────                             ──────────
mongo-secret.yaml  ──sops --encrypt──►  mongo-secret.enc.yaml  ──git commit──► repo
                          │
                    age private key                    at deploy time:
                    (stored securely,               sops --decrypt ──► plain Secret
                     e.g. ~/.age/key.txt,                                    │
                     AWS KMS, GCP KMS)                              kubectl apply
```

- You encrypt/decrypt files locally with `sops` + an `age` key (or cloud KMS).
- The encrypted file is committed; the key is stored separately (never in git).
- Not tied to a specific cluster — same encrypted file deploys to dev, staging,
  and prod.

**Pros:** key is portable (same encrypted file across clusters), works with
cloud KMS (AWS KMS, GCP KMS, Azure Key Vault) for auto-rotation.  
**Cons:** more moving parts, decryption must happen at deploy time (CI/CD
needs access to the key).

### Which to use when?

| Scenario | Recommendation |
|---|---|
| Learning, single local Minikube cluster | Sealed Secrets — simpler |
| Multiple clusters (dev/staging/prod) | SOPS + age or cloud KMS |
| CI/CD pipeline that deploys to multiple environments | SOPS + cloud KMS |
| GitOps with Argo CD | Either — both are supported natively |

**We implement Sealed Secrets in this lesson** (it runs entirely on Minikube).
SOPS is covered conceptually so you know it exists and when to choose it.

---

## How Sealed Secrets works — in detail

### The asymmetric encryption model

When you install the Sealed Secrets Helm chart, the controller generates a
**4096-bit RSA key pair** and stores it as a Secret in `kube-system`:

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
```

The **public key** is freely downloadable (anyone can encrypt with it).  
The **private key** never leaves the cluster — only the controller can decrypt.

```
kubeseal workflow:
1. kubeseal fetches the controller's public key
2. kubeseal encrypts each Secret data value with that public key + AES-256-GCM
3. The result is a SealedSecret resource (safe to commit)
4. When you apply the SealedSecret, the controller decrypts it and creates
   a normal Secret object inside the cluster
5. Your pods consume the normal Secret as usual — nothing changes in their specs
```

The encrypted value also includes the **namespace + name** of the target Secret,
so a SealedSecret for `default/mongo-secret` cannot be applied to
`production/mongo-secret` — this prevents secret re-use attacks.

---

## What changes in this lesson

### Before (insecure)

```
lessons/01-k8s-basics/mongo-secret.yaml   ← base64 in git — insecure
```

Pods reference `mongo-secret` directly. The Secret is applied by running
`kubectl apply -f mongo-secret.yaml` from your local machine.

### After (lesson 8)

```
lessons/08-config-secrets/mongo-sealed-secret.yaml  ← encrypted — safe to commit
```

You apply the `SealedSecret`. The controller creates the `Secret` object
automatically. Your pods still reference `mongo-secret` by name — **no changes
to pod specs**.

The original `mongo-secret.yaml` with real credentials should NOT be committed
to git. Keep it local (or in a password manager) and add it to `.gitignore` if
needed. The sealed secret file is what goes into git.

---

## Installation — Two methods

The Sealed Secrets controller can be installed via **Helm** (recommended) or raw
**kubectl manifests** (alternative for WSL2 or Helm credential issues).

### Method 1: Helm (recommended)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install sealed-secrets bitnami/sealed-secrets \
  -n kube-system \
  -f lessons/08-config-secrets/sealed-secrets-values.yaml
```

### Method 2: Raw manifests (alternative)

If Helm fails with credential service errors (common in WSL2):

```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/controller.yaml
```

Both methods produce an identical controller. Choose the one that works for your
environment. Verify the pod is running (same for both):

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=sealed-secrets
# Wait for 1/1 Running before proceeding
```

---

## Step-by-step setup

### Step 1 — Install the controller (choose one method above)

See **Installation — Two methods** section above. Pick Helm or raw manifests
based on your environment.

### Step 2 — Install the kubeseal CLI

**Option A: Dynamic version** (recommended if network is stable):

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest \
  | grep '"tag_name"' | sed 's/.*"v\([^"]*\)".*/\1/')

echo $KUBESEAL_VERSION  # Verify it prints a version like 0.27.0

curl -sSL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" \
  | tar xz kubeseal

sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

**Option B: Hardcoded version** (fallback if API fetch fails):

```bash
KUBESEAL_VERSION="0.27.0"

curl -sSL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz" \
  | tar xz kubeseal

sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version
```

If the version is outdated, check [GitHub releases](https://github.com/bitnami-labs/sealed-secrets/releases).

### Step 3 — Seal the mongo secret

Make sure your cluster is running and the controller is ready.

```bash
kubeseal \
  --format yaml \
  --namespace default \
  < lessons/01-k8s-basics/mongo-secret.yaml \
  > lessons/08-config-secrets/mongo-sealed-secret.yaml
```

The resulting file looks like this (values are real ciphertext, not base64):

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mongo-secret
  namespace: default
spec:
  encryptedData:
    mongo-root-password: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...  # real RSA+AES ciphertext
    mongo-root-username: AgCH+oW5GD+fJhPfmfNPCfLj3fwOm...
  template:
    metadata:
      name: mongo-secret
      namespace: default
```

This file is **safe to commit to git**.

### Step 4 — Apply the SealedSecret and verify

```bash
# Delete the old plain Secret if it exists
kubectl delete secret mongo-secret --ignore-not-found

# Apply the SealedSecret
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml

# The controller should immediately decrypt and create the Secret
kubectl get secret mongo-secret
# NAME           TYPE     DATA   AGE
# mongo-secret   Opaque   2      3s

# Verify the data is correct
kubectl get secret mongo-secret -o jsonpath='{.data.mongo-root-username}' | base64 -d
```

### Step 5 — Confirm pods still work

The Secret name hasn't changed (`mongo-secret`), so all existing pod specs
that reference it will work without modification. Deploy the stack normally.

---

## Key rotation workflow

When you need to change the Mongo password:

```bash
# 1. Update mongo-secret.yaml locally with the new password (do NOT commit this file)
# 2. Re-seal it
kubeseal --format yaml --namespace default \
  < lessons/01-k8s-basics/mongo-secret.yaml \
  > lessons/08-config-secrets/mongo-sealed-secret.yaml

# 3. Apply the new SealedSecret
kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml

# 4. Commit and push the new sealed secret (the plaintext file stays local)
git add lessons/08-config-secrets/mongo-sealed-secret.yaml
git commit -m "chore(lesson08): rotate mongo credentials"
git push
```

---

## Backing up the controller key — CRITICAL

If you delete the Minikube cluster and recreate it, the controller generates a
new key pair. Your existing sealed secrets become **permanently undecryptable**.

Back up the key before any destructive cluster operation:

```bash
kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml
```

**Do NOT commit this file to git.** Store it in a password manager or secrets
vault. To restore after a cluster rebuild:

```bash
kubectl apply -f sealed-secrets-master-key.yaml
kubectl rollout restart deployment sealed-secrets -n kube-system
```

---

## SOPS + age — brief practical overview

If you later move to SOPS, the workflow is:

```bash
# Install age and sops
brew install age sops    # or apt, snap, etc.

# Generate a key pair (do this once; keep the private key safe)
age-keygen -o ~/.config/sops/age/keys.txt

# Encrypt a secret file in place
SOPS_AGE_RECIPIENTS=$(age-keygen -y ~/.config/sops/age/keys.txt) \
  sops --encrypt --age $SOPS_AGE_RECIPIENTS \
  lessons/01-k8s-basics/mongo-secret.yaml \
  > lessons/08-config-secrets/mongo-secret.sops.yaml

# Decrypt and apply at deploy time
sops --decrypt lessons/08-config-secrets/mongo-secret.sops.yaml | kubectl apply -f -
```

The `mongo-secret.sops.yaml` file is safe to commit. The private key in
`~/.config/sops/age/keys.txt` is not — protect it like a password.

---

## Common mistakes

### 1. Committing the plain `mongo-secret.yaml` alongside the sealed secret

The most common mistake. Add `*-secret.yaml` to `.gitignore` for lessons that
don't use Sealed Secrets, or at minimum never commit files with real credentials.

### 2. Applying a SealedSecret to the wrong namespace

A SealedSecret encrypted for `namespace: default` will be rejected if you
`kubectl apply` it in `namespace: production`. The namespace is baked into the
encryption. Re-seal with `--namespace production` if you need it in a different
namespace.

### 3. Losing the controller key and having no backup

After a `minikube delete && minikube start`, the new controller generates a new
key. Old SealedSecrets are permanently lost. Always back up the key before
destroying a cluster.

### 4. Re-sealing when the controller key has rotated

Sealed Secrets supports key rotation: the controller generates a new key
periodically but keeps old keys to decrypt existing secrets. New secrets are
always encrypted with the current key. Run `kubeseal --re-encrypt` to update
all existing SealedSecrets to the current key after rotation.

### 5. Using `--controller-name` / `--controller-namespace` wrong

If the controller was installed under a different name or namespace, `kubeseal`
won't find it automatically. Pass the flags explicitly:

```bash
kubeseal --controller-name sealed-secrets --controller-namespace kube-system ...
```

---

## Debugging exercise

### Scenario: SealedSecret applied to wrong namespace → Secret not created

1. Create a test namespace:
   ```bash
   kubectl create namespace staging
   ```

2. Apply the SealedSecret (which was sealed for `namespace: default`) to the
   wrong namespace:
   ```bash
   kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml -n staging
   ```

3. Check whether the Secret was created:
   ```bash
   kubectl get secret mongo-secret -n staging
   # Error: NotFound
   ```

4. Diagnose:
   ```bash
   # Check the SealedSecret's own status
   kubectl describe sealedsecret mongo-secret -n staging
   # Look for: Events — error decrypting key: no key could decrypt secret
   # OR: the resource was created but the controller rejected it
   ```

5. Fix: delete the incorrectly applied SealedSecret and apply it to `default`:
   ```bash
   kubectl delete sealedsecret mongo-secret -n staging
   kubectl apply -f lessons/08-config-secrets/mongo-sealed-secret.yaml -n default
   kubectl get secret mongo-secret -n default
   ```

---

## Review questions

1. Why is a Kubernetes Secret with base64-encoded data still insecure when
   committed to a git repository?

2. Explain how Sealed Secrets uses asymmetric encryption. Who holds the private
   key, and what does that mean if the cluster is deleted?

3. You need to deploy the same application to dev, staging, and prod clusters.
   With Sealed Secrets you would need to seal the secret three times (once per
   cluster). How does SOPS + age solve this differently?

4. A colleague applies a SealedSecret to the cluster but the corresponding
   plain Secret is never created. Walk through the diagnostic steps.

5. What should you do before running `minikube delete`? Why?
