# Kubernetes Learning Roadmap — Node.js/NestJS Production Platform

This repo is a hands-on curriculum for taking a basic MongoDB + Node/NestJS
Kubernetes deployment to a production-grade setup. Each topic lives in its own
`lessons/NN-topic-slug/` folder with a matching doc in `docs/`.

## Starting point

Lesson 1 ([`docs/01-k8s-basics.md`](01-k8s-basics.md),
`lessons/01-k8s-basics/`) documents the original hand-written manifests this
repo started from:
- `mongo.yaml` — MongoDB Deployment + ClusterIP Service
- `mongo-config.yaml` — ConfigMap with the Mongo service DNS name
- `mongo-secret.yaml` — Secret with base64-encoded Mongo credentials
- `webapp.yaml` — webapp Deployment (public demo image) + NodePort Service

Original commit: `9012c3e initial files from Youtube learning` on `main`.

## Repo layout convention

- `lessons/NN-topic-slug/` — the actual Kubernetes manifests/charts/configs
  for that lesson (e.g. `lessons/01-k8s-basics/`, `lessons/02-helm/`).
- `docs/NN-topic.md` — the lesson write-up (see "Docs layout convention" below).
- `docs/commands/NN-topic-commands.md` — the command reference for that lesson.

## Branching convention

- `main` stays as the "known good, currently deployed" state.
- **Lessons 1 and 2** (`k8s-basics`, `helm`) were committed directly to `main`
  — they document work that was already done/verified before this repo's
  learning-mentor workflow started, so there was no in-progress state worth
  branching for.
- **From lesson 3 onward**, one branch per lesson: `lesson/NN-topic-slug`,
  e.g. `lesson/03-kustomize`. Merge to `main` only when the lesson is
  verified working end-to-end (`kubectl`/`helm` checks pass). Use a normal
  merge commit or fast-forward — your call — but don't squash away the
  intermediate learning commits, they are the point of this repo.
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
- `feat(lesson03): add Kustomize base + overlays`
- `docs(lesson03): add Kustomize lesson notes and review questions`
- `fix(lesson06): add missing memory limit on webapp container`

## Topic order and folders

| # | Topic | Folder / Branch | Status |
|---|-------|--------|--------|
| 1 | K8s Basics (Deployments, Services, ConfigMaps, Secrets) | `lessons/01-k8s-basics/` (on `main`) | done |
| 2 | Helm | `lessons/02-helm/` (on `main`) | done |
| 3 | Kustomize | `lesson/03-kustomize` | done |
| 4 | Ingress | `lesson/04-ingress` | done |
| 5 | Health Probes (liveness/readiness/startup) | `lesson/05-health-probes` | in progress |
| 6 | Resource Requests & Limits | `lesson/06-resource-limits` | in progress |
| 7 | Persistent Volumes (Mongo data) | `lesson/07-persistent-volumes` | in progress |
| 8 | Config & Secrets Management (Sealed Secrets / SOPS) | `lesson/08-config-secrets` | in progress |
| 9 | RBAC & Security (PSA, NetworkPolicy, non-root) | `lesson/09-rbac-security` | in progress |
| 10 | Horizontal Pod Autoscaler | `lesson/10-hpa` | not started |
| 11 | Monitoring & Logging (Prometheus/Grafana, Loki) | `lesson/11-monitoring-logging` | not started |
| 12 | CI/CD (GitHub Actions) | `lesson/12-ci-cd` | not started |
| 13 | GitOps (Argo CD) | `lesson/13-gitops-argocd` | not started |
| 14 | Deployment Strategies (Rolling/Blue-Green/Canary) | `lesson/14-deployment-strategies` | not started |

## Per-lesson deliverable checklist

Every lesson in this course produces, in this order:
1. Concept overview (why/when/trade-offs/common mistakes) in `docs/NN-topic.md`.
2. Practical implementation — actual files changed in this repo.
3. `kubectl`/`helm` verification commands you actually ran, with expected output noted.
4. A debugging exercise (something you deliberately break and diagnose).
5. A git commit (or a few) with the convention above.
6. 3–5 review questions before moving on.

## Docs layout convention

- `docs/NN-topic.md` — the lesson itself: concept, why/when/trade-offs,
  common mistakes, what files changed and why, the debugging exercise,
  review questions. Written assuming you're seeing the topic for the first
  time — no prior knowledge of that specific tool/feature assumed.
- `docs/commands/NN-topic-commands.md` — every command you ran for that
  lesson, in order, each with a one-line explanation of what it does and
  what output to expect. Kept separate from the lesson doc so you can use
  it as a quick copy-paste cheat sheet without re-reading the theory.

## Environment

- Local cluster: Minikube on WSL2 (Docker driver), context `minikube`.
- Tools already available: `kubectl` v1.36 client, `helm` v3.16, `kustomize`
  (bundled in kubectl), `docker`.
- Cloud is only used if a topic genuinely requires it (none of the 14 topics
  currently do — Argo CD, Ingress, HPA, monitoring all run fine on Minikube).

## Known gap to flag before topic 12 (CI/CD)

The webapp currently runs a public tutorial image
(`nanajanashia/k8s-demo-app:v1.0`), not your own NestJS source in this repo.
CI/CD (build + push your own image) needs actual application source. Decide
before lesson 12 whether to: (a) drop in your real NestJS app source here,
or (b) build a minimal placeholder NestJS app in this repo purely for the
CI/CD and GitOps lessons. Flagged again in that lesson's doc when we get there.
