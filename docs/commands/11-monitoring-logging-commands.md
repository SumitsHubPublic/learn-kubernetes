# Lesson 11 — Monitoring & Logging commands

**Prometheus** = metrics collection and time-series storage.
**Grafana** = dashboards and visualization.
**Loki** = log storage and search.
**Promtail** = log shipper to Loki.

Commands in order. Run from the repo root unless stated otherwise.

---

## 1. Create the monitoring namespace

```bash
kubectl apply -f lessons/11-monitoring-logging/namespace.yaml
```

Verify it exists:

```bash
kubectl get namespace monitoring
```

Expected output: a namespace named `monitoring` in `Active` state.

---

## 2. Add the Helm repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

This exposes the charts we need for Prometheus/Grafana and Loki.

---

## 3. Install the Prometheus + Grafana stack

```bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f lessons/11-monitoring-logging/prometheus-values.yaml
```

Verify the stack came up:

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

Expected output: Prometheus, Grafana, kube-state-metrics, and node-exporter
pods in `Running` or `Ready` state.

---

## 4. Open Grafana

The values file exposes Grafana via `NodePort` on port `30900` in Minikube.

```bash
kubectl get svc -n monitoring | grep grafana
```

Then open the UI:

- `http://<minikube-ip>:30900`
- or use port-forward locally:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Browse to `http://localhost:3000` and log in with:

- username: `admin`
- password: `admin`

> In a real cluster, change the admin password before exposing Grafana outside
> the internal network.

---

## 5. Check Prometheus targets

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open `http://localhost:9090/targets` and confirm the targets are `up`.

For a quick CLI check:

```bash
kubectl get servicemonitors -A
kubectl get podmonitors -A
```

If targets are down, the first place to inspect is the scrape endpoint and the
selector labels.

---

## 6. Install Loki + Promtail

```bash
helm upgrade --install loki \
  grafana/loki-stack \
  --namespace monitoring \
  -f lessons/11-monitoring-logging/loki-values.yaml
```

Check that Loki and Promtail are running:

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
kubectl get pods -n monitoring -l app.kubernetes.io/name=promtail
```

Expected output: Loki and Promtail pods should be in `Running` state.

---

## 7. Port-forward Loki for a health check

```bash
kubectl port-forward -n monitoring svc/loki 3100:3100
```

Then verify readiness:

```bash
curl http://localhost:3100/ready
```

Expected output: `ready` or an HTTP `200` response from the Loki server.

---

## 8. Inspect Promtail logs

```bash
kubectl logs -n monitoring daemonset/promtail --tail=50
```

This shows whether the log shipper is successfully reading pods and pushing them
into Loki.

If the daemonset name differs slightly in your Helm release, use:

```bash
kubectl get daemonset -n monitoring
```

Then target the correct pod set with `kubectl logs`.

---

## 9. Query log data from the Loki API

Once Promtail is shipping logs, you can query Loki directly:

```bash
curl "http://localhost:3100/loki/api/v1/query?query={namespace=\"default\"}"
```

This returns recent log entries from the default namespace.

> The exact label set depends on how your workloads are named and labeled.

---

## 10. Clean up lesson 11 resources

```bash
helm uninstall kube-prometheus-stack -n monitoring
helm uninstall loki -n monitoring
kubectl delete namespace monitoring
```

This removes the monitoring plane and its namespace entirely.

---

## 11. Optional: check the webapp / Mongo metrics path after the stack is live

If you later add metrics to the webapp (for example via a Micrometer or
Prometheus client library), the monitoring stack can scrape that app with a
ServiceMonitor or PodMonitor. That is the next production step after the basic
cluster telemetry is running and working.
