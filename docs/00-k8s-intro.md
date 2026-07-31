# Kubernetes for Beginners 🚀

> A beginner-friendly guide to learning Kubernetes from scratch.

---

# Table of Contents

1. What is Kubernetes?
2. Why Kubernetes?
3. Kubernetes Architecture
4. Kubernetes Objects
5. Setting Up Kubernetes
6. kubectl Basics
7. Deploying Your First Application
8. Services
9. ConfigMaps & Secrets
10. Volumes
11. Namespaces
12. Scaling Applications
13. Rolling Updates & Rollbacks
14. Troubleshooting
15. Useful Commands Cheat Sheet
16. Learning Roadmap
17. Recommended Resources

---

# 1. What is Kubernetes?

**Kubernetes (K8s)** is an open-source container orchestration platform used to automate the deployment, scaling, and management of containerized applications.

Imagine running hundreds of Docker containers across many servers. Kubernetes helps manage all of them automatically.

Without Kubernetes:

* Start containers manually
* Restart crashed containers manually
* Scale applications manually
* Configure networking manually

With Kubernetes:

* Automatically restarts failed containers
* Scales applications with one command
* Load balances traffic
* Performs rolling updates
* Manages storage and networking

---

# 2. Why Kubernetes?

Suppose your application receives 1 million users overnight.

Without Kubernetes:

* Add more servers manually
* Start new containers manually
* Configure networking manually

With Kubernetes:

```text
User Traffic
      │
      ▼
+------------------+
| Kubernetes       |
+------------------+
      │
 ┌────┼────┐
 ▼    ▼    ▼
Pod1 Pod2 Pod3
```

Simply run:

```bash
kubectl scale deployment nginx --replicas=3
```

Kubernetes handles the rest.

---

# 3. Kubernetes Architecture

```text
                    Kubernetes Cluster

          +-------------------------------+
          |         Control Plane          |
          |------------------------------- |
          | API Server                     |
          | Scheduler                      |
          | Controller Manager             |
          | ETCD (Cluster Database)        |
          +---------------+---------------+
                          |
        -----------------------------------------
        |                                       |
+---------------+                     +---------------+
| Worker Node 1 |                     | Worker Node 2 |
|---------------|                     |---------------|
| kubelet       |                     | kubelet       |
| kube-proxy    |                     | kube-proxy    |
| Pods          |                     | Pods          |
+---------------+                     +---------------+
```

## Control Plane

The Control Plane makes decisions for the cluster.

### API Server

The entry point to Kubernetes.

Every command such as:

```bash
kubectl get pods
```

is sent to the API Server.

---

### Scheduler

Determines which worker node should run a newly created Pod.

---

### Controller Manager

Ensures the cluster stays in the desired state.

Example:

Desired Pods = 3

Current Pods = 2

Controller creates another Pod automatically.

---

### ETCD

A distributed key-value database.

Stores:

* Cluster configuration
* Secrets
* Deployments
* Nodes
* Services

Think of it as Kubernetes' brain.

---

## Worker Node

A machine where applications run.

Components:

* kubelet
* kube-proxy
* Container Runtime
* Pods

---

### kubelet

Communicates with the API Server.

Responsibilities:

* Starts Pods
* Monitors Pods
* Reports Pod status

---

### kube-proxy

Handles networking.

Routes traffic to Pods.

---

### Container Runtime

Runs containers.

Examples:

* containerd
* CRI-O

---

# 4. Kubernetes Objects

Everything in Kubernetes is an object.

## Pod

Smallest deployable unit.

```text
+------------------+
| Pod              |
|------------------|
| Docker Container |
+------------------+
```

Example:

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx

spec:
  containers:
  - name: nginx
    image: nginx
```

Create it:

```bash
kubectl apply -f pod.yaml
```

---

## Deployment

A Deployment manages Pods.

Features:

* Rolling updates
* Self-healing
* Scaling
* Rollbacks

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 2

  selector:
    matchLabels:
      app: nginx

  template:

    metadata:
      labels:
        app: nginx

    spec:

      containers:

      - name: nginx
        image: nginx
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

---

## ReplicaSet

Ensures a specific number of Pods are always running.

If one Pod crashes:

```text
ReplicaSet
     │
     ▼
Create New Pod
```

Usually managed automatically by Deployments.

---

## Service

Pods have temporary IP addresses.

A Service provides a stable endpoint.

```text
Client
   │
   ▼
Service
   │
 ┌─┴─┐
 ▼   ▼
Pod Pod
```

Types:

* ClusterIP
* NodePort
* LoadBalancer
* ExternalName

---

# 5. Setting Up Kubernetes

## Option 1 — Minikube

Great for beginners.

Install:

```bash
minikube start
```

Check:

```bash
kubectl get nodes
```

---

## Option 2 — Kind

Runs Kubernetes clusters inside Docker.

```bash
kind create cluster
```

---

## Option 3 — Docker Desktop

Enable Kubernetes in the Docker Desktop settings.

---

# 6. kubectl Basics

List Pods:

```bash
kubectl get pods
```

List Nodes:

```bash
kubectl get nodes
```

List Deployments:

```bash
kubectl get deployments
```

Describe Pod:

```bash
kubectl describe pod POD_NAME
```

View Logs:

```bash
kubectl logs POD_NAME
```

Delete Pod:

```bash
kubectl delete pod POD_NAME
```

Apply YAML:

```bash
kubectl apply -f deployment.yaml
```

Delete Resource:

```bash
kubectl delete -f deployment.yaml
```

---

# 7. Deploy Your First Application

Create deployment:

```bash
kubectl create deployment nginx --image=nginx
```

Check:

```bash
kubectl get deployments
```

View Pods:

```bash
kubectl get pods
```

Expose:

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

Access:

```bash
minikube service nginx
```

---

# 8. Services

## ClusterIP

Internal communication only.

```text
Frontend ---> Backend
```

---

## NodePort

Exposes application on every node.

```text
Browser
    │
NodeIP:30000
    │
    ▼
Pod
```

---

## LoadBalancer

Used in cloud providers.

Creates an external load balancer.

---

# 9. ConfigMaps & Secrets

## ConfigMap

Stores configuration.

Example:

```yaml
apiVersion: v1

kind: ConfigMap

metadata:
  name: app-config

data:
  ENV: production
```

---

## Secret

Stores sensitive data.

```yaml
apiVersion: v1

kind: Secret

metadata:
  name: db-secret

type: Opaque

data:
  password: cGFzc3dvcmQ=
```

> **Note:** Secret values are Base64-encoded, not encrypted by default. Use encryption at rest and appropriate access controls for production environments.

---

# 10. Volumes

Containers lose data when restarted.

Volumes provide persistent storage.

```text
Container
     │
     ▼
 Persistent Volume
```

Common resources:

* emptyDir
* PersistentVolume (PV)
* PersistentVolumeClaim (PVC)

---

# 11. Namespaces

Namespaces separate resources.

Example:

```bash
kubectl get pods --namespace=dev
```

Create:

```bash
kubectl create namespace dev
```

---

# 12. Scaling Applications

Scale to five Pods:

```bash
kubectl scale deployment nginx --replicas=5
```

Verify:

```bash
kubectl get pods
```

---

# 13. Rolling Updates & Rollbacks

Update image:

```bash
kubectl set image deployment/nginx nginx=nginx:latest
```

Check rollout:

```bash
kubectl rollout status deployment/nginx
```

Rollback:

```bash
kubectl rollout undo deployment/nginx
```

---

# 14. Troubleshooting

View Pods:

```bash
kubectl get pods
```

Describe Pod:

```bash
kubectl describe pod POD_NAME
```

Logs:

```bash
kubectl logs POD_NAME
```

Execute inside a Pod:

```bash
kubectl exec -it POD_NAME -- /bin/sh
```

Check cluster events:

```bash
kubectl get events
```

---

# 15. kubectl Cheat Sheet

## Inspection Commands

| Command                                     | Description             |
| ------------------------------------------- | ----------------------- |
| `kubectl get all`                           | View everything (Pods, Services, Deployments, ReplicaSets, StatefulSets) |
| `kubectl get pods`                          | List Pods               |
| `kubectl get nodes`                         | List Nodes              |
| `kubectl get svc`                           | List Services           |
| `kubectl get deployments`                   | List Deployments        |
| `kubectl describe pod POD_NAME`             | Pod details             |
| `kubectl logs POD_NAME`                     | Pod logs                |
| `kubectl exec -it POD_NAME -- /bin/sh`      | Open shell in Pod       |
| `kubectl apply -f file.yaml`                | Create/update resources |
| `kubectl scale deployment APP --replicas=5` | Scale application       |
| `kubectl rollout status deployment/APP`     | Watch deployment        |
| `kubectl rollout undo deployment/APP`       | Roll back deployment    |

## Resource Cleanup & Removal

| Command                                     | Description             |
| ------------------------------------------- | ----------------------- |
| `kubectl delete pod POD_NAME`               | Delete a specific Pod   |
| `kubectl delete deployment DEPLOYMENT_NAME` | Delete a Deployment (cascades to Pods) |
| `kubectl delete svc SERVICE_NAME`           | Delete a Service        |
| `kubectl delete -f file.yaml`               | Delete all resources defined in a YAML file |
| `kubectl delete -f <directory>/`            | Delete all resources in a directory |
| `kubectl delete all --all`                  | Delete ALL resources in current namespace ⚠️ |
| `kubectl delete all --all -n NAMESPACE`     | Delete ALL resources in a specific namespace ⚠️ |

> **⚠️ Warning:** Commands with `--all` are destructive. Use with caution in production environments.

---

# 16. Learning Roadmap

```text
Docker
   │
   ▼
Containers
   │
   ▼
Pods
   │
   ▼
Deployments
   │
   ▼
ReplicaSets
   │
   ▼
Services
   │
   ▼
ConfigMaps
   │
   ▼
Secrets
   │
   ▼
Volumes
   │
   ▼
Namespaces
   │
   ▼
Ingress
   │
   ▼
Helm
   │
   ▼
Monitoring
   │
   ▼
Production Kubernetes
```

---

# 17. Recommended Resources

### Official Documentation

* Kubernetes Documentation
* Kubernetes Concepts
* Kubernetes Tasks

### Practice Platforms

* Minikube
* Kind
* Play with Kubernetes
* Killercoda Kubernetes Labs

### Beginner Projects

* Deploy an NGINX web server
* Deploy a Node.js application
* Create a multi-tier application
* Use ConfigMaps and Secrets
* Add Persistent Volumes
* Configure an Ingress
* Deploy with Helm

---

# Best Practices for Beginners

* Learn Docker before Kubernetes.
* Write YAML manually instead of relying only on generators.
* Use `kubectl describe` and `kubectl logs` frequently while debugging.
* Practice on a local cluster before moving to cloud environments.
* Learn one Kubernetes object at a time and understand how they relate.

---

# What's Next?

After completing this guide, you should be comfortable with:

* ✅ Kubernetes architecture
* ✅ Pods
* ✅ Deployments
* ✅ Services
* ✅ Scaling
* ✅ Rolling updates
* ✅ ConfigMaps
* ✅ Secrets
* ✅ Volumes
* ✅ Namespaces
* ✅ Basic troubleshooting

From here, continue learning:

1. Ingress
2. Helm
3. StatefulSets
4. DaemonSets
5. Jobs & CronJobs
6. RBAC
7. Network Policies
8. Monitoring with Prometheus & Grafana
9. GitOps (Argo CD / Flux)
10. Production cluster operations

Happy learning! 🎉
