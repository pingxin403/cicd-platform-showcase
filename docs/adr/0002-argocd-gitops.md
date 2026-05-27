# ADR-0002: ArgoCD over Push-Based Deploys

- **Status**: Accepted
- **Context**: Kubernetes-deployed services across dev/staging/prod

## Context

Two viable deployment models for K8s services:

1. **Push-based**: CI runs `kubectl apply` / `helm upgrade` after build
2. **Pull-based (GitOps)**: a controller in-cluster reconciles desired state from a Git repo (ArgoCD, Flux)

Push-based is faster to bootstrap and has simpler debugging ("the failure is in the CI run"). GitOps adds infrastructure but resolves an entire class of problems.

## Decision

Use **ArgoCD** with an `applications` repo separate from each service's source repo:

```
service-source-repo  →  CI  →  builds image  →  bumps tag in apps-repo  →  PR
                                                                              ↓
                                                              ArgoCD reconciles
                                                                              ↓
                                                                       cluster
```

## Why GitOps

- **Audit trail without effort** — every cluster change is a git commit; `git log` is the audit log
- **Drift detection** — ArgoCD detects manual edits and either flags or auto-heals
- **Approval ergonomics** — promotion from staging to prod is a PR review, not a CI button
- **Disaster recovery** — `git clone` + `kubectl apply -f argocd-bootstrap.yaml` restores cluster state

## Why ArgoCD specifically (not Flux)

- Better UI for non-DevOps stakeholders to inspect rollout state
- ApplicationSet resource elegantly handles "one definition, many environments"
- Better Helm + Kustomize integration for the typical patterns we use

Flux is a defensible alternative; the choice is preference, not correctness.

## Consequences

### Required investments

- Apps-repo structure: `envs/{dev,staging,prod}/<service>/` per environment per service
- Image-tag bump automation in CI (small custom action or `argocd-image-updater`)
- ArgoCD itself running in a "control" namespace, monitored separately

### What changes for developers

- "Deploy" = "merge a PR to apps-repo" — sometimes confusing on day 1
- Rollback = `git revert` (or revert PR) — ArgoCD reconciles back
- Hot-fix path documented separately (since direct `kubectl apply` defeats GitOps)

### What changes for ops

- Drift alerts go to a Slack channel instead of nobody
- Cluster bootstrap from scratch is now a documented procedure, not tribal knowledge

## Anti-patterns to avoid

- **Mixing push and pull**: do not let CI `kubectl apply` to the same namespace ArgoCD manages. Pick one per namespace.
- **Per-service apps-repos**: the cross-service env-promotion path becomes painful. Keep one apps-repo per cluster (or per environment-group).
- **Inline secrets**: secrets must come from Sealed Secrets / External Secrets / Vault. Never raw values in apps-repo.

## Related

- [ADR-0004](0004-canary-with-slo-gate.md) — how Flagger plugs into the GitOps flow
