# ADR-0003: Istio Header-Routing Lanes for Multi-PR Staging

- **Status**: Accepted
- **Context**: shared staging cluster, multiple PRs needing parallel validation

## Context

A team with N concurrent PRs wants to validate each in a "near-prod" environment. Two common approaches:

1. **Per-PR namespace**: spin up a full namespace clone per open PR, tear down on merge
2. **Per-PR routing lane**: one namespace, many `Deployment` revisions, route based on header/cookie

Per-PR namespace is conceptually clean but:
- Namespace explosion (10 active PRs × 6 services = 60 deployments)
- Cross-service traffic patterns become complicated to wire (each PR's frontend needs to talk to that PR's backend)
- DB schema migration per namespace gets expensive

## Decision

Use **Istio + header routing**: deploy each PR's services into the shared staging namespace with version labels, configure `VirtualService` to route based on `X-PR-Lane` header.

```
client → Istio gateway →
   header X-PR-Lane=PR-123 → service:v-PR-123
   header X-PR-Lane=PR-456 → service:v-PR-456
   no header               → service:main
```

Concretely:

1. CI on PR-123 deploys `frontend:v-PR-123`, `backend:v-PR-123` etc. into `staging` namespace
2. Istio `VirtualService` is a generated resource that lists all active lanes
3. Tester sets `X-PR-Lane: PR-123` (browser extension or curl flag) → only PR-123's revisions handle traffic
4. Database is shared; PR-specific schema changes are gated behind feature flags / shadow tables

A garbage-collection job tears down lanes whose PR is closed or stale > 7 days.

## Why this works

- **Real cross-service communication** — PR-123's frontend talks to PR-123's backend through real network paths, not mocks
- **Real DB** — shared database catches schema-compatibility issues at PR time
- **Cheap to spin up** — 1 lane = N deployments, not N namespaces
- **Cheap to compare** — A/B testing two implementation strategies = two header values

## Why not feature flags only

Feature flags are excellent for product flags, but:
- Force every conditional into application code (debugging gets harder)
- Don't help with infrastructure-level changes (e.g. "trying a new sidecar config")
- Don't help with validating *image-level* differences (e.g. "does this Go upgrade break anything")

Lanes complement feature flags — they validate the artefact, flags validate the behaviour.

## Consequences

- **Routing config drift**: VirtualService listing all lanes can grow unwieldy — automated via a controller / template, not edited by hand.
- **Database schema discipline**: any schema change must be backward-compatible while at least 2 lanes are live. This is good hygiene anyway (matches expand/contract migration pattern).
- **Header injection** required: testers / E2E tests must set the lane header. Browser dev extension works; for E2E, a Playwright test fixture sets it per session.
- **Observability needs the header too**: Prometheus / Jaeger labels must include the lane so dashboards are filterable.

## Validation

- Open 3 unrelated PRs; verify each isolates correctly (set header, hit endpoint, see correct version in response).
- Open a PR that breaks `main`; verify `main` lane is unaffected.
- Close all PRs; verify garbage-collection cleans up.

## Related

- Companion idea for prod: [ADR-0004](0004-canary-with-slo-gate.md) — Flagger uses similar Istio primitives but for time-based progressive rollout instead of header-based concurrent rollout.
