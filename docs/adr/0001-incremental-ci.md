# ADR-0001: Dynamic Incremental CI over Matrix-Everything

- **Status**: Accepted
- **Context**: monorepos (cuckoo, cuckoo-echo) where N services share a CI graph

## Context

A naive monorepo CI runs every service's lint + test + build + image-publish on every PR. As N grows, CI time grows linearly even when a PR touches one service.

Two failure modes appear:

1. **Slow PR feedback** — single-line README change triggers 18-minute CI
2. **Flaky-test amplification** — flaky test in unrelated service blocks unrelated PR

## Decision

Use a **change-detection step** at the start of CI to compute a per-service "needs rebuild" bit, then matrix the rest of the pipeline against the resulting set.

Concretely:

1. First job (`detect-changes`) uses `dorny/paths-filter` (or equivalent) on `paths` per service
2. Subsequent jobs use `if: needs.detect-changes.outputs.<svc> == 'true'`
3. Build artifacts are content-hashed and cached at `actions/cache`; reuse across PRs targeting the same base SHA
4. Image publish is also gated — unchanged services are not re-tagged

A reference implementation is at [`docs/workflows-samples/incremental-ci.yml`](../workflows-samples/incremental-ci.yml).

## Why not matrix-everything

Matrix is simpler to write but:
- **Cost grows linearly** with services × OS × language combinations
- **No semantic skip** — even if a Go service didn't change, its Java sibling builds again
- **Cache thrashing** — global caches conflict; per-service caches not addressable cleanly

## Why not Bazel / pants for true incremental

Bazel solves this elegantly, but:
- High onboarding cost for a 5–10-service monorepo
- Spring Boot / Maven idioms don't translate without effort
- The 70/30 rule: change-detection in Actions YAML solves 70% of the pain at 5% of the migration cost

For >50 services or >100 engineers, revisit Bazel/pants.

## Consequences

- **CI time**: in cuckoo, single-service PRs go from "build all 7 services" to "build 1 + verify deps" — local measurement on the cuckoo monorepo dropped a single-service-change PR from ~12 min to ~3 min on warm cache. Numbers will vary by repo.
- **First-time PR cost**: cold-cache PR is full duration; this is acceptable.
- **Edge case — shared library change**: when `cuckoo-common` (shared module) changes, all dependent services must rebuild. The detection step handles this with explicit dependency declarations in `paths-filter` config.
- **Drift risk**: if a service's `paths-filter` rule misses a real dependency, that service may skip CI for a PR that broke it. Mitigated by a nightly full-CI run on `main`.

## Validation

- Run a known-noop PR (e.g. fix a typo in only one service's README); expect only that service's pipeline.
- Run a shared-library PR; expect all dependents to rebuild.
- Compare wall-clock CI time before/after enabling.

## Related

- [ADR-0002](0002-argocd-gitops.md) — what happens after CI succeeds
- Sample workflow: [`incremental-ci.yml`](../workflows-samples/incremental-ci.yml)
