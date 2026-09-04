# 108 Ting Ecosystem — Docs

Ecosystem-level design, decision, deployment, and readiness docs for the
108 Ting Ecosystem. Private — contains internal infra topology, payment/identity
contracts, and active-work status.

## Standards

- [`ENGINEERING_STANDARD_AI_AUTHORED.md`](ENGINEERING_STANDARD_AI_AUTHORED.md) —
  how we structure and gate code now that agents write it and at most two people
  supervise. Seven rules, each with the incident that produced it, plus the
  measured quality baseline for `pos108-core`.
- [`DEPLOYMENT_STANDARD_K3S.md`](DEPLOYMENT_STANDARD_K3S.md) — k3s deployment standard.
- [`CI_STANDARD_AND_WORKFLOWS.md`](CI_STANDARD_AND_WORKFLOWS.md) — ecosystem CI standard, runner allocation strategy, and workflow catalog for remaining repos.
- [`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md) — what "ready" means.
- [`VERSIONING_STANDARD.md`](VERSIONING_STANDARD.md) — what is actually running in
  front of a customer: the `1.MINOR.PATCH` number, the channel, and the runtime
  surface every app must expose. Binding on every app.

> Note: several docs here link to `ENGINEERING_CONSTITUTION.md`, which has never
> existed in this repo or anywhere in the org — a dangling reference, not a moved
> file. Either write it or repoint those links.
