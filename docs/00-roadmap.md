# Kubernetes Learning Roadmap — Node.js/NestJS Production Platform

This repo is a hands-on curriculum for taking a basic MongoDB + Node/NestJS
Kubernetes deployment to a production-grade setup. Each topic below becomes
one branch and one (or a few) small commits, with a matching doc in `docs/`.

## Starting point (already done)

Captured in [`00-baseline-review.md`](00-baseline-review.md):
- `mongo.yaml` — MongoDB Deployment + ClusterIP Service
- `mongo-config.yaml` — ConfigMap with the Mongo service DNS name
- `mongo-secret.yaml` — Secret with base64-encoded Mongo credentials
- `webapp.yaml` — webapp Deployment (public demo image) + NodePort Service

Commit: `9012c3e initial files from Youtube learning` on `main`.

## Branching convention

- `main` stays as the "known good, currently deployed" state.
- One branch per lesson: `lesson/NN-topic-slug`, e.g. `lesson/01-helm`.
- Merge to `main` only when the lesson is verified working end-to-end
  (`kubectl`/`helm` checks pass). Use a normal merge commit or fast-forward —
  your call — but don't squash away the intermediate learning commits, they
  are the point of this repo.
- If a lesson needs a throwaway experiment (e.g. deliberately breaking
  something for a debugging exercise), do it in a sub-branch off the lesson
  branch: `lesson/NN-topic-slug/debug-<scenario>`, and delete it once done.

## Commit message convention

Conventional-commit style, scoped to the lesson number:

```
<type>(lessonNN): <short summary>

<optional body: what changed and why>
```

Types used in this course: `feat` (new capability), `docs` (docs-only),
`fix` (bug found during a lesson), `refactor` (restructure without behavior
change), `chore` (tooling).

Examples:
- `feat(lesson01): convert manifests to a Helm chart`
- `docs(lesson01): add Helm lesson notes and review questions`
- `fix(lesson05): add missing memory limit on webapp container`

## Topic order and branch names

| # | Topic | Branch | Status |
|---|-------|--------|--------|
| 1 | Helm | `lesson/01-helm` | in progress |
| 2 | Kustomize | `lesson/02-kustomize` | not started |
| 3 | Ingress | `lesson/03-ingress` | not started |
| 4 | Health Probes (liveness/readiness/startup) | `lesson/04-health-probes` | not started |
| 5 | Resource Requests & Limits | `lesson/05-resource-limits` | not started |
| 6 | Persistent Volumes (Mongo data) | `lesson/06-persistent-volumes` | not started |
| 7 | Config & Secrets Management (Sealed Secrets / SOPS) | `lesson/07-config-secrets` | not started |
| 8 | RBAC & Security (PSA, NetworkPolicy, non-root) | `lesson/08-rbac-security` | not started |
| 9 | Horizontal Pod Autoscaler | `lesson/09-hpa` | not started |
| 10 | Monitoring & Logging (Prometheus/Grafana, Loki) | `lesson/10-monitoring-logging` | not started |
| 11 | CI/CD (GitHub Actions) | `lesson/11-ci-cd` | not started |
| 12 | GitOps (Argo CD) | `lesson/12-gitops-argocd` | not started |
| 13 | Deployment Strategies (Rolling/Blue-Green/Canary) | `lesson/13-deployment-strategies` | not started |

## Per-lesson deliverable checklist

Every lesson in this course produces, in this order:
1. Concept overview (why/when/trade-offs/common mistakes) in `docs/NN-topic.md`.
2. Practical implementation — actual files changed in this repo.
3. `kubectl`/`helm` verification commands you actually ran, with expected output noted.
4. A debugging exercise (something you deliberately break and diagnose).
5. A git commit (or a few) with the convention above.
6. 3–5 review questions before moving on.

## Environment

- Local cluster: Minikube on WSL2 (Docker driver), context `minikube`.
- Tools already available: `kubectl` v1.36 client, `helm` v3.16, `kustomize`
  (bundled in kubectl), `docker`.
- Cloud is only used if a topic genuinely requires it (none of the 13 topics
  currently do — Argo CD, Ingress, HPA, monitoring all run fine on Minikube).

## Known gap to flag before topic 11 (CI/CD)

The webapp currently runs a public tutorial image
(`nanajanashia/k8s-demo-app:v1.0`), not your own NestJS source in this repo.
CI/CD (build + push your own image) needs actual application source. Decide
before lesson 11 whether to: (a) drop in your real NestJS app source here,
or (b) build a minimal placeholder NestJS app in this repo purely for the
CI/CD and GitOps lessons. Flagged again in that lesson's doc when we get there.
