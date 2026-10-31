# Lesson 11 — Monitoring & Logging (Prometheus/Grafana, Loki)

**Monitoring** = time-series metrics about cluster health and workload behavior.
**Logging** = stream of structured/unstructured events from pods, nodes, and
system components.

## What problem does this solve?

By lesson 10 the stack is scaling and the app is largely healthy. But we still
have blind spots:

- we can see `kubectl get pods`, but not whether the cluster is saturating
  CPU/memory over time
- we cannot tell whether the webapp is returning slow responses or elevated
  error rates
- when a pod crashes we have logs only on the node, not a central place to
  search them
- we have no alerting path when a deployment stays unhealthy or a node loses
  capacity

This lesson adds the standard Kubernetes observability stack:

1. **Prometheus** — scrapes metrics from kubelet, cAdvisor, node exporters,
   service monitors, and custom exporters.
2. **Grafana** — dashboards and visualization on top of Prometheus.
3. **Loki** — log storage and queries for Kubernetes container logs.
4. **Promtail** — agent that tails pod logs and pushes them to Loki.

Together they create the basic production signal needed to debug, monitor, and
alert on real system behavior.

---

## Why metrics and logs are different

Metrics and logs answer different questions:

- Metrics are numeric, time-series, and great for trend analysis, alerting, and
  dashboards.
- Logs are event streams, great for debugging a specific failure or sequence of
  actions.

A pod can look healthy in Grafana while still returning 500s due to a logic bug
in application code. Logs are usually the fastest way to inspect the specific
error path; metrics tell you it is happening repeatedly and at what rate.

---

## The monitoring architecture in this lesson

```text
┌──────────────────────┐
│ Kubernetes objects   │
│ Deployments, Nodes,  │
│ Pods, Services       │
└──────────┬───────────┘
           │ Prometheus scrapes
           ▼
┌──────────────────────┐      read/query      ┌──────────────────────┐
│ Prometheus           │ ───────────────────► │ Grafana              │
│ - kube-state-metrics │                     │ - dashboards         │
│ - node-exporter      │                     │ - graphs             │
│ - service monitors   │                     │ - alert panels       │
└──────────┬───────────┘                     └──────────────────────┘
           │
           │
           ▼
┌──────────────────────┐
│ Loki                 │
│ - log index + store  │
└──────────┬───────────┘
           │ query
           ▼
┌──────────────────────┐
│ Promtail             │
│ - tails pod logs     │
│ - ships to Loki      │
└──────────────────────┘
```

This is a typical “metrics + logs + dashboards” stack used in Kubernetes.
Prometheus is the center of the metrics plane; Loki handles the logs plane.

---

## What Prometheus gives us

Prometheus stores data as time series keyed by:

- metric name
- labels (`namespace`, `pod`, `container`, `job`, `instance`, etc.)
- sample timestamp and value

Examples of useful metrics in a cluster:

- `up` — whether a target is scraping successfully
- `container_cpu_usage_seconds_total` — CPU consumed by containers
- `container_memory_usage_bytes` — memory usage
- `kube_pod_status_phase` — pod lifecycle state
- `kube_deployment_status_replicas_available` — availability

Prometheus also supports the PromQL query language, which is what you use to
build alerts and dashboards, such as:

```promql
sum(rate(container_cpu_usage_seconds_total{pod=~"webapp-.*"}[5m])) by (pod)
```

That query gives CPU usage per webapp pod over the last 5 minutes.

---

## What Grafana gives us

Grafana is the visualization layer on top of Prometheus and Loki. It lets you:

- build dashboards for CPU, memory, pod restarts, request latency, and error
  rate
- query both metrics and logs in the same UI
- create alerts that page or notify a team when a threshold is crossed

In production you usually create a small set of dashboards for:

- cluster overview
- node health
- deployment health
- application latency/errors
- log drill-through

---

## What Loki gives us

Loki is a logs backend designed for Kubernetes and container workloads. It is
optimized for log indexing and searches by labels such as:

- `namespace`
- `pod`
- `container`
- `app`
- `release`

It is not a full general-purpose event store like Elasticsearch; it is purpose-
built to handle container logs efficiently and cheaply.

The query pattern usually looks like:

```logql
{namespace="default", app="webapp"} |= "ERROR"
```

This surfaces only the relevant log lines from the webapp pods and filters them
for error messages.

---

## Why a log shipper is necessary

Kubernetes does not automatically centralize logs into a cluster-wide backend.
Each pod writes to stdout/stderr; the container runtime may keep logs on disk,
but they are not easy to search across every node.

That is why a log agent such as Promtail or Fluent Bit exists:

1. tail container log files or stdout streams
2. attach labels (`namespace`, `pod`, `container`)
3. push them to Loki
4. allow Grafana or Loki queries to find the relevant events

Without a log agent, you are forced to run `kubectl logs` on a specific pod by
hand, which does not scale.

---

## The gap this stack doesn't close: APM and distributed tracing

Prometheus/Grafana/Loki gives you infrastructure metrics and raw logs, but it
does not answer "which function call inside the webapp made this request slow"
or "which downstream service caused this error to cascade". That's the job of
**Application Performance Monitoring (APM)** and **distributed tracing**:

- APM instruments application code (via an agent or SDK) to capture per-
  transaction timing, DB query duration, error stack traces, and throughput
  broken down by endpoint/route.
- Distributed tracing follows a single request as it crosses process/service
  boundaries (webapp → Mongo, webapp → another microservice), producing a
  waterfall of spans so you can see exactly where time was spent.

Nothing installed so far in this lesson does this automatically — Prometheus
only sees what your app chooses to expose as metrics, and Loki only sees log
lines your app already writes. Getting APM/tracing means either instrumenting
the app yourself (e.g. OpenTelemetry SDK + Jaeger/Tempo, still self-hosted), or
adopting a commercial SaaS APM platform. This is where New Relic fits in.

---

## Where New Relic fits in

**New Relic** is a commercial, SaaS observability platform. Instead of running
Prometheus/Grafana/Loki yourself, you install a New Relic agent (or the New
Relic Kubernetes integration) and it ships metrics, logs, traces, and APM data
to New Relic's hosted backend — you never operate the storage or query engine.

### What New Relic adds on top of this lesson's stack

| Capability | Prometheus/Grafana/Loki (this lesson) | New Relic |
|---|---|---|
| Infra metrics (CPU/mem/node/pod) | Yes, via node-exporter + kube-state-metrics | Yes, via the New Relic Kubernetes integration (`nri-kubernetes`) |
| Logs | Yes, via Promtail → Loki | Yes, via New Relic's log forwarder or Fluent Bit plugin |
| Dashboards | Yes, via Grafana | Yes, built-in dashboards + custom NRQL dashboards |
| Alerting | Yes (Alertmanager, not covered yet in this repo) | Yes, built-in alert policies/notifications |
| **APM (code-level transaction tracing)** | No — needs manual OpenTelemetry instrumentation | Yes, out of the box via language agents (Node.js, Java, Python, etc.) |
| **Distributed tracing across services** | No — needs Jaeger/Tempo + manual instrumentation | Yes, automatic with APM agents |
| Storage/scaling operations | You own it (PVCs, retention, sharding Loki/Prometheus at scale) | Fully managed by New Relic |
| Cost model | Infra cost only (compute/storage you run) | Usage-based SaaS billing (ingested data volume + users) |

### How the New Relic Kubernetes integration works

```text
┌──────────────────────┐
│ Kubernetes cluster   │
│ - nodes, pods        │
│ - kube-state-metrics │
└──────────┬───────────┘
           │ scraped by
           ▼
┌──────────────────────┐      pushes data      ┌──────────────────────┐
│ newrelic-infrastructure│ ───────────────────► │ New Relic (SaaS)     │
│ (DaemonSet agent)     │                       │ - metrics, logs      │
│ + nri-kubernetes      │                       │ - APM, tracing       │
│ + New Relic APM agent │                       │ - dashboards, alerts │
│   (in-app, per pod)   │                       │                      │
└──────────────────────┘                       └──────────────────────┘
```

Two separate pieces are typically installed:

1. **Cluster-level integration** (`newrelic-infrastructure` Helm chart) — a
   DaemonSet that reports node/pod/cluster metrics and events, similar in
   scope to node-exporter + kube-state-metrics combined.
2. **APM agent** — a language-specific library added to the application itself
   (e.g. `newrelic` npm package for a Node.js/NestJS app) that instruments HTTP
   handlers, DB calls, and outbound requests, then reports transaction traces.

For this repo's webapp, only the cluster-level integration would apply today,
since the app runs a public demo image with no New Relic agent baked in. Real
APM data would require adding the New Relic Node.js agent to the app's own
source — noted as a future step once lesson 12 addresses building our own
image.

### When to reach for New Relic instead of (or alongside) self-hosted OSS

- You want APM/tracing without running Jaeger/Tempo and instrumenting
  OpenTelemetry by hand.
- You don't want to operate storage, retention, and scaling for Prometheus/Loki
  yourself — especially across multiple clusters.
- You need a single pane of glass across many clusters/clouds without
  federating multiple self-hosted Prometheus instances.
- Trade-off: recurring SaaS cost that scales with data volume, and your
  telemetry data leaves your infrastructure.

Prometheus/Grafana/Loki and New Relic are not mutually exclusive — some teams
run both: Prometheus/Grafana for infra dashboards they fully control, and New
Relic (or another APM vendor) purely for application-level tracing that would
be expensive to build and maintain in-house.

---

## What changes in this lesson

### New files

| File | Purpose |
|---|---|
| `lessons/11-monitoring-logging/namespace.yaml` | Monitoring namespace for the observability stack |
| `lessons/11-monitoring-logging/prometheus-values.yaml` | Helm values for kube-prometheus-stack (Prometheus + Grafana) |
| `lessons/11-monitoring-logging/loki-values.yaml` | Helm values for Loki + Promtail |

This lesson is intentionally additive: no existing app manifests are changed.
The monitoring stack sits in a dedicated `monitoring` namespace and watches the
cluster without altering the webapp or Mongo workloads themselves.

---

## Common mistakes

### 1. Installing the stack in the wrong namespace

If you install Helm releases in the default namespace instead of a dedicated
`monitoring` namespace, you lose separation and make cleanup harder. Keep the
stack isolated for easier upgrades and RBAC boundaries.

### 2. Forgetting to install `--create-namespace`

`helm install` will fail if the namespace does not exist. The commands doc uses
`--create-namespace` to make the install path easy and repeatable.

### 3. Expecting every workload to expose Prometheus metrics automatically

Prometheus does not magically scrape every deployment. It works by selecting a
set of targets via ServiceMonitor, PodMonitor, or static configs. A pod that
does not expose `/metrics` or does not have a matching ServiceMonitor will be
absent from the metrics plane.

### 4. Treating Loki as if it were Prometheus

Prometheus stores numeric time series; Loki stores logs. They are complementary,
not interchangeable. Monitoring dashboards usually query Prometheus, while log
exploration uses Loki.

### 5. Skipping the verification commands

`kubectl get pods -n monitoring` and `kubectl get svc -n monitoring` are the
fastest way to tell whether the stack is alive before you open Grafana or run
queries.

### 6. Expecting a SaaS APM agent to replace infra metrics for free

Installing a New Relic (or Datadog, etc.) agent alongside Prometheus/Loki
means you're now shipping the same infrastructure metrics to two places and
paying for both. Decide deliberately which platform owns which signal (e.g.
Prometheus/Grafana for infra, New Relic only for APM/tracing) instead of
duplicating everything by default.

---

## Debugging exercise

### Scenario: Prometheus is up, but Grafana shows no data

This is a common issue when the stack installs successfully but the scrape
config is not selecting the correct targets.

**Break it:**

1. Install the stack using the lesson files.
2. Confirm the pods are all `Running`.
3. Open Grafana and check whether the default Prometheus data source is there.
4. Then check the Prometheus targets page in Grafana or via the Prometheus UI.

**Diagnose it:**

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open `http://localhost:9090/targets` and look for `up == 0` or `TargetDown`.
The usual causes are:

- the metrics endpoint is not reachable
- the target does not expose a Prometheus scrape port
- the ServiceMonitor/PodMonitor selector does not match the workload
- the namespace or label selector is wrong

**Fix it:**

- verify the service selector and target labels
- confirm the workload is exposing a port with metrics
- recheck the target configuration and reload Prometheus

If a target is missing entirely, the problem is usually not the cluster itself,
but the scrape configuration or endpoint exposure.

---

## Lesson 11 goal

The goal is not to build a huge enterprise observability platform in one pass.
The goal is to install the baseline Kubernetes monitoring and logging stack and
understand the roles of each component:

- Prometheus answers: “what is happening and how much?”
- Grafana answers: “how do I visualize it?”
- Loki answers: “where are the relevant log lines?”

That gives you the minimum mental model needed for production debugging and
alerting.

---

## Review questions

1. Why are metrics and logs complementary instead of interchangeable?
2. Why is Prometheus usually paired with Grafana instead of used directly for
   dashboards?
3. What is the purpose of a log shipper such as Promtail or Fluent Bit?
4. Why do we keep the observability stack in its own `monitoring` namespace?
5. What is the first thing you check when Grafana shows no data, even though
   the Prometheus pods are running?
6. What does APM/distributed tracing give you that Prometheus + Loki cannot,
   even once both are fully working?
7. What are the two separate pieces you'd install to get full New Relic
   coverage of this repo's stack, and why does only one of them apply today?

---

## Next lesson preview

The next course step after telemetry is CI/CD. Once the app and cluster are
observable, the next major production concern is automating image builds, tests,
and deployments so that changes can reach Kubernetes reliably without manual
commands.
