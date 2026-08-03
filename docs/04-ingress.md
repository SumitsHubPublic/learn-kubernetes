# Lesson 4 — Ingress

## What problem does Ingress solve?

After lesson 1 you exposed the webapp with a **NodePort** service. That works
in Minikube for quick demos, but it has real problems in production:

| Problem | Why it matters |
|---|---|
| Each service needs its own node port (30000–32767) | Hard to remember, ugly URLs, firewall rules multiply |
| No hostname routing | Can't serve `api.example.com` and `app.example.com` from one IP |
| No path routing | Can't send `/api/*` to one service and `/` to another |
| No TLS termination in the service itself | Every service would have to handle its own certs |
| NodePort exposes the node IP directly | Not safe in production; you want a single public entry point |

**Ingress** is a Kubernetes API object that defines HTTP/HTTPS routing rules.
A separate component called an **Ingress Controller** watches for those objects
and configures a real load-balancer or reverse proxy (nginx, Traefik, AWS ALB,
etc.) accordingly.

---

## Ingress Controller vs Ingress Resource — the critical distinction

This is where most beginners get confused:

```
┌─────────────────────────────────────────────────────────────┐
│  Kubernetes cluster                                         │
│                                                             │
│  ┌──────────────────────┐        ┌───────────────────────┐  │
│  │  Ingress Controller  │  reads │  Ingress Resource     │  │
│  │  (a running Pod)     │◄───────│  (your YAML file)     │  │
│  │  e.g. nginx, Traefik │        │  rules, hosts, paths  │  │
│  └──────────┬───────────┘        └───────────────────────┘  │
│             │ proxies to                                     │
│  ┌──────────▼───────────┐                                   │
│  │  webapp-service      │  ClusterIP                        │
│  │  (port 3000)         │                                   │
│  └──────────┬───────────┘                                   │
│             │                                               │
│  ┌──────────▼───────────┐                                   │
│  │  webapp Pod(s)       │                                   │
│  └──────────────────────┘                                   │
└─────────────────────────────────────────────────────────────┘
        ▲
        │  HTTP request for webapp.local
   (browser / curl)
```

- **Ingress Controller** = the software doing the actual proxying. It is
  a Deployment running inside your cluster (usually in the
  `ingress-nginx` namespace). You install it once.
- **Ingress Resource** = the routing rules you write per-app in YAML.
  It only has effect if a controller is present and watching.

> Analogy: the controller is nginx itself; the Ingress resource is a
> `server {}` block in nginx.conf. Writing the nginx.conf block without
> installing nginx does nothing.

---

## Why ClusterIP instead of NodePort now?

Once an Ingress Controller handles external traffic, the backend services
only need to be reachable **inside** the cluster — ClusterIP is enough.
Keeping services as NodePort would still work but:
- It leaves unnecessary ports open on every node.
- It creates confusion about which entry point is "the real one."
- In production clouds the node ports are often blocked by security groups anyway.

**Rule of thumb:** Services behind an Ingress should be ClusterIP. Only the
Ingress Controller's own Service needs to be LoadBalancer (or NodePort on bare
metal/Minikube).

---

## What changed in this lesson

### `lessons/04-ingress/webapp-service.yaml` — NodePort → ClusterIP

```yaml
# Before (lesson 1):
spec:
  type: NodePort
  ports:
    - port: 3000
      targetPort: 3000
      nodePort: 30007

# After (lesson 4):
spec:
  type: ClusterIP
  ports:
    - port: 3000
      targetPort: 3000
```

`nodePort: 30007` is gone. The service is no longer reachable from outside the
cluster directly. The Ingress Controller will reach it via its cluster-internal
DNS name `webapp-service.default.svc.cluster.local:3000`.

### `lessons/04-ingress/webapp-ingress.yaml` — new Ingress resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: webapp-service
            port:
              number: 3000
```

Key fields explained:

| Field | What it does |
|---|---|
| `ingressClassName: nginx` | Tells K8s which controller to use. Minikube's addon registers class `nginx`. Without this, the Ingress is silently ignored. |
| `host: webapp.local` | Only requests with `Host: webapp.local` match this rule. Requests to the raw IP are rejected with 404. |
| `path: /` with `pathType: Prefix` | Matches any path starting with `/` — i.e., all paths. `Exact` would only match exactly `/`. |
| `backend.service.port.number: 3000` | Forward to port 3000 on `webapp-service`. Must match the service's `port`, not `targetPort`. |

---

## How to apply

### Step 1 — Enable the Ingress addon in Minikube

```bash
minikube addons enable ingress
```

Wait ~60 seconds for the controller pod to become `Running`:

```bash
kubectl get pods -n ingress-nginx
```

Expected output (all pods `Running` or `Completed`):
```
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-xxxxx        0/1     Completed   0          2m
ingress-nginx-admission-patch-xxxxx         0/1     Completed   0          2m
ingress-nginx-controller-xxxxxxxxx-xxxxx    1/1     Running     0          2m
```

### Step 2 — Apply the base stack (if not already running)

The Ingress lesson adds to the same stack from lesson 1. Apply the base
manifests first if the cluster is fresh:

```bash
kubectl apply -f lessons/01-k8s-basics/mongo-secret.yaml
kubectl apply -f lessons/01-k8s-basics/mongo-config.yaml
kubectl apply -f lessons/01-k8s-basics/mongo.yaml
kubectl apply -f lessons/01-k8s-basics/webapp.yaml
```

### Step 3 — Apply the lesson 4 manifests

```bash
# Replace the NodePort service with ClusterIP
kubectl apply -f lessons/04-ingress/webapp-service.yaml

# Create the Ingress resource
kubectl apply -f lessons/04-ingress/webapp-ingress.yaml
```

### Step 4 — Verify the Ingress got an address

```bash
kubectl get ingress webapp-ingress
```

Expected output:
```
NAME             CLASS   HOSTS          ADDRESS        PORTS   AGE
webapp-ingress   nginx   webapp.local   192.168.49.2   80      30s
```

The `ADDRESS` column shows Minikube's IP. If it stays empty after ~60 seconds,
the controller isn't running or `ingressClassName` is wrong — see debugging
exercise below.

Get Minikube's IP:
```bash
minikube ip
# e.g. 192.168.49.2
```

### Step 5 — Add a local DNS entry

Ingress routing is host-based: the nginx controller checks the `Host:` header.
You need `webapp.local` to resolve to Minikube's IP on your machine.

On Linux/macOS (WSL2 — edit the WSL hosts file):
```bash
echo "$(minikube ip) webapp.local" | sudo tee -a /etc/hosts
```

Verify:
```bash
curl http://webapp.local
```

Expected: the webapp HTML page.

---

## Common mistakes

### 1. Missing `ingressClassName`

If you omit `ingressClassName: nginx`, Kubernetes will not assign the Ingress
to any controller. The `ADDRESS` column stays blank forever and requests to
`webapp.local` return a connection refused or 404.

Fix:
```bash
kubectl describe ingress webapp-ingress
# Look for: "Default backend: <default>"
# and no "Controller" annotation — that's the tell.
```

### 2. Wrong port in the Ingress backend

The `port.number` in the Ingress must match the `port` (not `targetPort`) of
the Service. If you put `80` instead of `3000`, nginx will try to connect to
`webapp-service:80`, which isn't listening → **502 Bad Gateway**.

### 3. `Host:` header mismatch

If you `curl http://192.168.49.2` (the IP directly) without a `Host` header,
nginx returns 404 because no rule matches. You must `curl http://webapp.local`
or pass `-H "Host: webapp.local"`.

### 4. `/etc/hosts` in WSL2 vs Windows

In WSL2, `/etc/hosts` inside the Linux environment is separate from the Windows
`C:\Windows\System32\drivers\etc\hosts`. Curl and kubectl from the WSL terminal
use the WSL one. Your Windows browser uses the Windows one. Edit both if you
want to test from a browser in Windows.

### 5. Forgetting to wait for the controller pod

`minikube addons enable ingress` just installs the controller manifests.
The controller pod itself needs to pull its image and start. Applying the
Ingress resource before the controller pod is Running means the Ingress gets
no address until the controller starts up and re-syncs.

---

## Debugging exercise

### Scenario: wrong backend port causes 502

1. Edit `lessons/04-ingress/webapp-ingress.yaml` and change `number: 3000`
   to `number: 80`.
2. Apply: `kubectl apply -f lessons/04-ingress/webapp-ingress.yaml`
3. Run: `curl http://webapp.local` — you should get **502 Bad Gateway**.
4. Diagnose:

```bash
# Check the Ingress config looks correct syntactically
kubectl describe ingress webapp-ingress

# Check if the service endpoint actually has port 3000
kubectl get endpoints webapp-service
# Expected: shows 172.17.x.x:3000 — NOT :80

# Confirm the service spec
kubectl get svc webapp-service -o yaml | grep -A5 ports
```

The endpoints show `:3000` but the Ingress is sending traffic to `:80` — that's
your mismatch.

5. Fix: revert `number` back to `3000` and re-apply.
6. Verify: `curl http://webapp.local` returns the webapp again.

---

## Review questions

1. What is the difference between an Ingress Controller and an Ingress Resource?
   Why does having only the resource (but no controller) do nothing?

2. You have two microservices: `api-service` (port 8080) and `frontend-service`
   (port 3000). Write an Ingress YAML that routes `/api/*` to `api-service` and
   `/` to `frontend-service`, both on the same host `myapp.local`.

3. Why did we change the webapp Service from NodePort to ClusterIP in this
   lesson? What would happen if we left it as NodePort?

4. What does `pathType: Prefix` mean versus `pathType: Exact`? Give an example
   of a path that would match `/api` with `Prefix` but not with `Exact`.

5. A colleague applies a new Ingress but `kubectl get ingress` shows the
   `ADDRESS` column empty after 5 minutes. Walk through the steps you would
   take to diagnose this.
