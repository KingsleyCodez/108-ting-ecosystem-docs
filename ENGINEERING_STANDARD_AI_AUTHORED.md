# Engineering standard — software authored by agents

**Status:** active, 2026-07-30. Supersedes repo-layout reasoning inherited from
docs written for a human-authored codebase (`M4-DECISION-HYBRID`,
`M4-POS-KERNEL-EXTRACTION`, `M4-POS-SHARED-CRATES-MIGRATION`). Those remain
useful as history; they are no longer the yardstick.

## Why a new standard

Nobody hand-writes this codebase any more. Almost all of it is written by agents,
supervised by at most two people.

Most received wisdom about how to structure a codebase was written for teams of
humans, and its reasons do not survive the change:

| Classic reason to split a repo | Does it still hold? |
|---|---|
| Team ownership / Conway's law | No — there are no teams |
| Smaller repo, less for a person to hold in their head | No — an agent greps; it never reads the whole tree |
| Independent release cadence per team | No — one supervisor releases everything |
| Fewer merge conflicts between people | No — work is serialised through review |
| Scoping code review | No — review is per-diff, not per-repo |

What *does* change is the failure mode. A human carries "I still have to bump the
rev in the other repo" across days. An agent's context ends with its session; the
second half of a two-repo change is simply never done.

That is not a hypothesis. Every consolidation in this org has failed in exactly
that place, and never in the code itself:

- **`pos-kernel` / `pos-error` / `pos-infra` split (2026-07-24).** Phases 1–3
  (create the repos) and Phase 5 (migrate `108-platform-services`) shipped.
  **Phase 4 — migrate `pos108-core` — was never run.** Six days later core had
  moved on and `ORDER_CANCELLED` held two different values in two repos, with
  nothing able to detect it.
- **Wave 3 (`core` into the monorepo, PR #15).** Reverted because some core
  sub-crates still said `pos-error = { path = "../pos-error" }` after those
  crates moved out, so `cargo metadata` could not load the workspace. The fix was
  to write `{ workspace = true }`. The whole migration was reverted instead.
- **Image builds for all five services (PRs #20, #22).** `cargo build --locked`
  resolves the full lock including the private `pos-*` git deps; the build
  container had no git credential, and `rust:1-slim` has no `git` binary at all.
  Latent since the git-dep migration, and it broke *every* redeploy.

The pattern: agents are reliable inside one verifiable unit and unreliable across
seams. So the standard is about removing seams, not about tidiness.

---

## The seven rules

### 1. A repo is one atomic, self-verifiable unit of change

If a single logical change needs edits in two repos plus a version bump, the
boundary is wrong. Merge them.

A change must be provable green by one command in one checkout. If proving it
requires publishing a rev first, the boundary is wrong.

### 2. The compiler is the only reviewer that never forgets

Anything duplicated where no tool can compare the copies **will** drift, silently
and without a date.

Never keep two copies of the same code with only prose asking them to agree.

> Evidence: three verbatim forks of `pos-kernel`; `ORDER_CANCELLED` with two
> values; `customer` and `loyalty` carrying `PayrollRunId` and `LadyDrinkRunId`
> they never used, inherited by copy.

### 3. Building must require no credential

Every private git dependency adds a secret that can expire, be absent, or be
missing from a container — and each one fails at a different layer.

> Evidence, all real: `SHARED_CRATES_TOKEN` per consumer; BuildKit
> `--mount=type=secret,id=gh_token`; `rust:1-slim` shipping no `git`;
> `cargo-deny-action` running in a container with no git credential; the
> `ghcr-pull` secret expiring silently in staging *and* production.

A crate you own should be a path dependency. If it cannot be, that is a finding,
not a design.

### 4. Verification runs in one command, offline

An agent that cannot verify cheaply will guess, or will report success it did not
observe.

> Done right: `.sqlx` committed with `SQLX_OFFLINE = "true"` in
> `.cargo/config.toml` — `cargo check` works with no database.
> Done wrong: a build that needs a PAT before it can resolve dependencies.

### 5. Split only for a different language, or a genuinely independent deploy with no shared code

Not for ownership. Not because a name sounds like a layer.

- **Correctly separate:** the frontends — `admin` (Next), `orders`/`store`
  (Solid), `sell` (SolidStart), Flutter dashboards.
- **Correctly together:** Rust backends that share crates — one workspace, one
  lockfile, path deps.

Before extracting anything, measure who uses what. `pos-infra` was extracted as a
whole crate because it was *named* infra; only 3 of its 20 modules (563 of 2,786
lines) were ever shared, and the remaining 65% forced the elaborate feature
gating and the dependency inversion onto `secrets-client` that followed.

### 6. Every step is verified against real state, and is safe to re-run

An agent resumes with no memory of what it already did. Exit codes are not state.

> Evidence: "the image built" and "the pod runs that image" are different facts —
> that gap bit twice in one night. A deploy gate reported success while making
> zero `kubectl` calls. A vault `PUT` returned `2xx` and upserted nothing.

Check the thing itself, and make the step idempotent so a re-run is always safe.

### 7. The supervisor is not the reviewer — a gate is

At the rate agents produce code, one person cannot read it all. Saying "the owner
reviews it" is not a control; it is a hope. Anything that matters must be checked
by something that runs on every change and cannot get tired or fall behind.

**Measured baseline, `pos108-core`, 2026-07-30** (~195,000 lines, 42 crates), so
future numbers mean something:

| Signal | Count |
|---|---:|
| `pub fn` total | 1,300 |
| `pub fn` referenced exactly once (candidate dead code) | 35 (2.7%) |
| `.rs` files with comments and zero code | 11 |
| `allow(clippy::too_many_arguments)` | 130 |
| `allow(clippy::type_complexity)` | 4 |
| `allow(dead_code)` / `allow(unused…)` | 8 / 1 |
| `TODO` / `FIXME` / `todo!()` / `unimplemented!` | 1 / 0 / 0 / 0 |

Read honestly: on the usual junk indicators this codebase is disciplined, not
sloppy. The problem is not that the numbers are bad — it is that **nobody was
measuring them**. `cargo-machete`, `cargo-udeps`, `jscpd` and `tokei` were not
installed anywhere in the fleet, so neither a claim that the code is clean nor a
claim that it is junk had any evidence behind it.

**The gate runs where the code is written, and CI is the floor.** This was got
wrong on the first attempt and is worth stating as part of the rule: the gate was
built as a CI job on the shared runner fleet, and its output was a comment on the
PR. But development happens on the author's machine and is pushed once finished,
so that comment arrives after the moment it could have changed anything, on a
server nobody is watching. A review nobody reads is not a control either.

So: the primary surface is a **pre-push hook on the author's machine**, printing
to the terminal of whoever wrote the code, before it leaves the machine
(`scripts/hygiene/install-hooks.sh`, once per clone). The identical checks stay in
CI as the backstop, because git hooks are not versioned and a clone without the
hook is the normal case, not a violation. Neither replaces the other: local is
where it gets read, CI is where it cannot be skipped.

**Two layers, and only one of them may block.** Everything decidable by a rule is
checked by a rule — same input, same verdict, every run, for free — and that layer
fails the build. What a rule cannot decide (is this abstraction wrong, does this
comment still describe the code, is this the same logic under another name) is
reviewed by a model, which **advises and never blocks**. An LLM is not
reproducible: the same diff can come back different on a re-run, and a gate that
answers differently each time is not a gate.

The advisory layer is not decoration. Run against the gate's own first
implementation it found six real defects in it, including a brace walker that
miscounted inside string literals and so *under-reported* the number the
deterministic layer blocks on — a false clean, which is the one failure mode a
ratchet cannot survive.

Both layers must also be honest about their own reach. A checker that crashes must
not report zero findings, and a claim that "the gate already covers X" must match
what the code actually covers, or the advisory layer is told to stay quiet about a
hole that is really there.

Required gates, in CI, on every PR (see [CI_STANDARD_AND_WORKFLOWS.md](file:///Users/yuth/Documents/108-Ting-Ecosystem/CI_STANDARD_AND_WORKFLOWS.md) for ecosystem workflow catalog and runner allocation rules):

- `cargo clippy --all-targets -- -D warnings` — already in place; keep it hard.
- **Unused dependencies** (`cargo-machete`) — an agent adding a crate it ends up
  not using is invisible to clippy. Not yet installed on the fleet, so as of
  2026-07-30 this one is reported as skipped rather than enforced; a check that is
  absent must say so rather than pass silently.
- **Duplication** (`jscpd`) — rule 2 says the compiler cannot see copies; this is
  what sees them.
- **Comment-only modules** — a module that declares nothing but promises
  behaviour in `//!` is worse than an absent module, because it reads as
  implemented. Fail the build.
- **A new `allow(...)` needs a reason on the line above it.** Today roughly half
  the 134 suppressions carry no explanation. A suppression without a reason is a
  lint that was silenced, not answered.
- **Track the baseline above per PR and fail on regression**, rather than
  discussing whether the codebase "feels" messy.

`too_many_arguments` at 130 sites is a genuine design signal and is now on the
record: it is not garbage code, but it is 130 functions that outgrew their
signature, and it should trend down rather than up.

---

## What this implies right now

Rules 1–4 all point the same way: **the Rust backends belong in one workspace.**
That is already the chosen direction ("the trajectory is MORE consolidation, not
less") — this standard says why, and what has to be true for it to hold.

Sequenced, because doing it in one step is how it failed before:

1. **Done (2026-07-30):** `pos-kernel` / `pos-error` / `pos-infra` folded back
   into `108-platform-services` as members, histories preserved, content unioned
   from both sides.
2. `pos108-core` drops its local copies and consumes them through the git source
   it already has for `secrets-client`.
3. Path-filtered CI plus `cargo-guppy determinator`, **before** step 4 — see the
   tripwire below.
4. `pos108-core` joins the workspace. Fix stale path deps to
   `{ workspace = true }`; do not revert on the first `cargo metadata` error.

### Tripwire, stated up front

Core is ~195,000 lines and 42 crates; `108-platform-services` is ~74,000. Merged,
that is ~270,000 lines and 60+ crates against five shared runners. Full-workspace
CI will not survive that, and the current setup deliberately tolerates
full-workspace builds because it is small enough today.

So step 3 is not optional polish — it is the precondition for step 4. Do not
start step 4 until affected-target CI is in place and measured.

### Budget is part of the standard

CI minutes are a shared, exhaustible resource: July 2026 closed at 2,701 of 3,000
included minutes, spent almost entirely by seven workflows still on GitHub-hosted
runners while the fleet was believed to be fully self-hosted. One Windows job was
1,030 of them, at a 2× billing multiplier.

- Self-hosted by default; a GitHub-hosted `runs-on` needs a reason in a comment.
- Every workflow has a `concurrency` group — a finite fleet queues instead of
  scaling out.
- Path-filter so an unchanged service claims no runner slot.
- A job that is cheap but `if: always()` still bills a rounded-up minute per run.

---

## Applying this to a proposal

Before creating a repo, extracting a crate, or adding a git dependency, answer:

1. Can one command in one checkout prove this change is correct?
2. Does anything except a comment keep the copies in agreement?
3. Does building it need a credential?
4. Measured, not assumed: who actually imports what, and how much?
5. Is the boundary a language or an independent deploy — or is it a name?
6. If an agent resumes mid-way with no memory, does it see true state?

7. Is a gate checking this on every change, or only a person who may not get to it?

Any "no" is the finding. Fix that, rather than proceeding and documenting the
consequence.
