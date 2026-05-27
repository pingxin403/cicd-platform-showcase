# cicd-platform-showcase

> Public docs-only mirror of CI/CD & release-governance practices applied across my microservice projects.
> 微服务 CI/CD 与发布治理实践的公开文档橱窗。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Source: across multiple repos — by invitation](https://img.shields.io/badge/source-across%20repos%20%E2%80%94%20by%20invitation-lightgrey)](#source-access--%E6%BA%90%E7%A0%81%E8%AE%BF%E9%97%AE)

---

## What this repo is / 这是什么

This repository is a **documentation showcase** of CI/CD and release-governance patterns I've designed and operated across my microservice projects. It contains:

- Architecture decision records (ADRs) for non-obvious choices
- Sanitized GitHub Actions workflow templates
- Release-governance design notes (canary, SLO/error-budget gating, rollback)
- A reading order for evaluators

It does **not** contain production secrets, internal cluster configurations, or company-specific playbooks. Real implementation lives in private repos and is shared on request.

本仓库是我在微服务项目里所做的 **CI/CD 与发布治理实践**的公开文档橱窗，包含 ADR、清理过的 GitHub Actions 模板、发布治理（金丝雀 / SLO 联动 / 回滚）的设计说明。**不含生产凭据、内部集群配置或公司特定 playbook**——真实实现位于私库，按需开放。

---

## Companion showcases / 配套橱窗

This is one corner of a three-repo showcase triangle covering my main practice areas:

- [**cuckoo-echo-showcase**](https://github.com/pingxin403/cuckoo-echo-showcase) — multi-tenant AI customer-service SaaS architecture
- **You are here** — CI/CD & release governance (this repo)
- [**observability-platform-showcase**](https://github.com/pingxin403/observability-platform-showcase) — observability platform (OTel, structured logging, alert layering, tail-sampling)

All three are docs-only and intentionally cross-reference where decisions span domains. For example: this repo's [ADR-0004 canary+SLO](docs/adr/0004-canary-with-slo-gate.md) consumes the SLO definitions from the observability showcase, and the deploy targets from cuckoo-echo's architecture.

---

## Project context / 项目背景

This showcase aggregates practices from:

- **[cuckoo](https://github.com/pingxin403/cuckoo)** (public) — polyglot Go/Java/TS monorepo, where the CI strategy was designed against real change patterns
- **cuckoo-echo** (private) — multi-tenant AI customer-service platform, where SLO/error-budget gating and Flagger-driven canary were validated
- Earlier work at previous employers (not represented here in source form)

Together they cover: GitHub Actions custom actions, dynamic incremental CI, ArgoCD GitOps, Istio multi-branch staging environments, Flagger canary, Terraform IaC, and SLO/error-budget-driven release gates.

---

## Capability map / 实践覆盖

| Layer | Practice | Status here |
|---|---|---|
| Pipeline | GitHub Actions custom actions, parallel build, dependency cache | [ADR-0001](docs/adr/0001-incremental-ci.md) + [sample workflow](docs/workflows-samples/) |
| Build | Dynamic incremental build (only changed services rebuild) | ADR-0001 |
| GitOps | ArgoCD declarative deploy + PR-driven rollout | [ADR-0002](docs/adr/0002-argocd-gitops.md) |
| Staging | Istio multi-branch lanes (header / cookie routing) for parallel feature validation | [ADR-0003](docs/adr/0003-multi-lane-staging.md) |
| Release | Flagger progressive canary tied to alert + SLO + error-budget | [ADR-0004](docs/adr/0004-canary-with-slo-gate.md) |
| Rollback | One-click + automatic rollback on canary failure | ADR-0004 |
| IaC | Terraform — versioned cloud infrastructure | covered in ADR-0002 |
| Security | SBOM generation, dependency-vulnerability gate | sample workflow only |

---

## Architecture decisions / 架构决策

| ID | Decision | Why it matters |
|---|---|---|
| [ADR-0001](docs/adr/0001-incremental-ci.md) | **Dynamic incremental CI over matrix-everything** | Cuts CI time when only one of N services changed |
| [ADR-0002](docs/adr/0002-argocd-gitops.md) | **ArgoCD over push-based deploys** | Audit trail, drift detection, PR-driven approval |
| [ADR-0003](docs/adr/0003-multi-lane-staging.md) | **Istio header-routing lanes over per-branch namespaces** | Avoids namespace explosion; multi-PR validation in shared cluster |
| [ADR-0004](docs/adr/0004-canary-with-slo-gate.md) | **Canary gating tied to alert + SLO + error budget** (not just metric thresholds) | Avoids "canary passes by luck"; ties release to user-visible reliability |

---

## Sample workflow templates / 工作流模板示例

Sanitized templates I use as starting points:

- [`docs/workflows-samples/incremental-ci.yml`](docs/workflows-samples/incremental-ci.yml) — dynamic change detection + parallel matrix
- [`docs/workflows-samples/sbom-and-scan.yml`](docs/workflows-samples/sbom-and-scan.yml) — SBOM generation + vulnerability gate
- [`docs/workflows-samples/preview-env.yml`](docs/workflows-samples/preview-env.yml) — per-PR preview environment lifecycle

> All templates are **scaffolding** — secrets, registry URLs, namespaces have been replaced with placeholders. Drop into a real repo and wire into your actual deploy targets.

---

## Suggested reading order / 建议阅读顺序

For a 10-minute walkthrough:

1. [`docs/adr/0001-incremental-ci.md`](docs/adr/0001-incremental-ci.md) — why incremental beats matrix at scale
2. [`docs/adr/0004-canary-with-slo-gate.md`](docs/adr/0004-canary-with-slo-gate.md) — release gating philosophy
3. [`docs/workflows-samples/incremental-ci.yml`](docs/workflows-samples/incremental-ci.yml) — see the strategy as code
4. [`docs/adr/0002-argocd-gitops.md`](docs/adr/0002-argocd-gitops.md) — what GitOps actually buys you
5. [`docs/adr/0003-multi-lane-staging.md`](docs/adr/0003-multi-lane-staging.md) — multi-PR concurrent validation

---

## Source access / 源码访问

Real CI workflows live in:

- **[cuckoo](https://github.com/pingxin403/cuckoo)** — public — `.github/workflows/` directory shows the dynamic-CI strategy in production-equivalent form
- **cuckoo-echo** (private) — full ArgoCD application set, Flagger canary configs, SLO-gated deploy automation

If you are evaluating this body of work, open an issue here or reach me directly. I'll grant time-boxed read access to the relevant private repo.

---

## Disclaimer / 免责声明

- **Single-author body of work.** The patterns described here come from real implementations, but no claim is made that these designs have been operated at production scale by a team of engineers across years.
- **Numbers are reproducible measurements**, not vendor benchmarks. Where I cite a CI-time reduction or canary success rate, the source is named (a specific repo + a specific dataset) — when in doubt, treat the number as a local baseline rather than a guaranteed SLA.
- **Templates are starting points**, not drop-in production solutions. Adapt to your secrets-management, identity, and registry conventions.

---

## License

[MIT](LICENSE) — applies to documentation in this showcase repository.

<!-- TODO: add a short post-mortem / before-after entry once a fresh real-world CI-time delta is measured -->
<!-- TODO: link to companion observability-platform-showcase once it ships -->
