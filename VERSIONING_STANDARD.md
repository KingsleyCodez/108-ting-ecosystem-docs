# 108 Ting Ecosystem — Versioning Standard

**Status:** binding on every app in the ecosystem, from the date this merges.
**Owner's instruction, 2026-09-01:** *"ระบบทุกตัวของ pos ต้องมีเวอร์ชั่นกำกับ … เพราะเราเริ่มจะขึ้น
โปรดักชั่นแล้ว … ให้ทุกแอฟปฏิบัติตามอย่างเคร่งครัด"*

Companion to `ENGINEERING_STANDARD_AI_AUTHORED.md` (how software is authored),
`PRODUCTION_READINESS_BAR.md` (what must be true before it ships) and
`DEPLOYMENT_STANDARD_K3S.md` (how it reaches the cluster). This one answers a single
question that none of those answer today:

> **What is actually running, right now, in front of this customer?**

---

## 1. Why this exists

Until now the ecosystem has had no shared answer. Measured across the repos on
2026-09-01, not inferred:

| repo | version it declares | what that number has ever meant |
|---|---|---|
| `pos108-terminal` | `0.1.94` | **real** — self-update compares it, so it must move |
| `pos108-core` | `0.1.0` | never moved; ships to production most days |
| `pos108-admin` | `0.1.0` | never moved |
| `pos108-orders` | `0.1.0` | never moved |
| `pos108-sell` | `0.1.0` | never moved |
| `pos108-store` | `0.0.0` | never even set |
| `park108` | `0.1.0` | never moved |
| `jh-api` | `1.0.0-alpha.5` | its own scheme, unrelated to the rest |

**One app out of eight versions for real**, and only because a machine depends on it.
Everywhere else the only true identifier is the image tag `sha-<short>`, which is
readable from `kubectl` and from nowhere else.

That was survivable while every shop belonged to the owner. It stops being survivable the
moment a stranger phones in with *"my screen is wrong"*, because the first question —
*which build are they on?* — has no answer that does not require cluster access.

Two live examples from the same day, both of which this standard would have caught:

- `tenants/tenant-admin` ran `sha-b7ba2bb` while its helm release still recorded
  `sha-117c2d4` from 32 days earlier. Nothing anywhere said so. (`pos108-admin#541`.)
- `pos108-admin` exposes **no version surface at all** — no `/api/version`, no health
  route. The only way to identify a running back office is to ask Kubernetes.

---

## 2. The number

**`MAJOR.MINOR.PATCH`, and MAJOR stays `1`.**

This is Brave's shape, adopted deliberately because this team already reads it daily on
its own fork. Verified on this machine: Brave reports `152.1.94.117` — Chromium major
`152` prefixed onto the product version `1.94.117`. The product number is `1.x.y` and has
been for years; the major is not a marketing lever.

| part | moves when | example |
|---|---|---|
| **MAJOR** | never, for now. Reserve it for a break so large the product is renamed | `1` |
| **MINOR** | each release train — a batch of work the team decides to ship together | `1.**95**.0` |
| **PATCH** | each build cut from that train, including fixes | `1.95.**3**` |

Rules:

- **A version is only real if something breaks when it is wrong.** `pos108-terminal` is the
  proof: its self-update compares versions, so `0.1.94` could never be a lie. Every other
  repo's number was decoration precisely because nothing read it. So: **the version must be
  emitted at runtime** (§3) and **asserted in CI** (§6) — otherwise this document produces
  eight more decorations.
- **Never hand-edit the number in a feature PR.** It moves in a release commit and nowhere
  else, or two branches will both claim `1.95.1`.
- `jh-api`'s `1.0.0-alpha.5` is grandfathered until its first non-alpha release, then it
  joins this scheme.

### Channels

Brave ships Nightly / Dev / Beta / Release as separate builds. We have two lanes and
should not pretend to four:

| channel | branch | who sees it |
|---|---|---|
| `staging` | `deploy/staging` | us |
| `release` | `release/prod` | customers |

The channel is **part of the reported version**, never part of the number:
`1.95.3 (release)`. A build that cannot say which lane it came from has caused a real
incident already — an admin build sat on staging while everyone believed it was on prod.

- **`channel` belongs to the deployment lane, not the image.** The exact same image artifact
  (`sha-<short>`) is promoted from staging to release. Therefore, `channel` must never be
  baked into Docker image build arguments or guessed from `NODE_ENV` (`NODE_ENV=production` is
  set in both staging and prod). It must be provided at deployment time via Helm values per
  namespace/lane.

---

## 3. Every app must answer for itself — mandatory

**A running app must be able to state its own version without cluster access.** This is the
rule that turns a number into a fact.

`pos108-core` already does it, and its shape is the ecosystem's reference:

```json
{"appVersion":"0.1.0","build":"sha-8464de0","builtAt":"2026-08-31T23:54:20Z",
 "database":true,"redis":true,"schemaVersion":"20261102000000","status":"ok"}
```

Every app exposes the same four identity fields, whatever else it adds:

| field | meaning | why it is not optional |
|---|---|---|
| `version` | `1.95.3` — the product number from §2 | what a human says on the phone |
| `build` | `sha-<short>` — the commit that produced the image | what an engineer greps |
| `builtAt` | ISO-8601 UTC | separates "old build" from "old deploy" |
| `channel` | `staging` \| `release` | see above |

Where it is served:

| kind of app | surface |
|---|---|
| backend service | `GET /health/ready` (already the convention) |
| web front end | `GET /api/version`, **and** rendered somewhere a user can read it |
| native / installed app | its About screen, and in its update check |

⚠️ **A web app must not report only its own version.** Brave's About screen shows Brave
*and* Chromium, because the browser is not the whole answer. Ours has the same shape: the
admin console is a bundle talking to a per-shop backend, so *"admin 1.95.3"* is half a
sentence. It must report the version of the API it is talking to as well —
`admin 1.95.3 · core 1.95.1 · shop 365`. Without the second half, a report of "the admin is
broken" cannot be placed on either side of the boundary.

### Missing or unknown values — strict rules (Owner decision 2026-09-02, #15 / #17)

1. **All four identity fields (`version`, `build`, `builtAt`, `channel`) must always be present.**
   Never omit a field when its value is unavailable (e.g. `if let Some(ch) = channel` omitting
   the key is forbidden; see `pos108-core#1114`). Omitting keys breaks typed API contracts and
   leaves callers unable to distinguish an unpopulated field from a legacy pre-standard build.
2. **The only legal representation of an unpopulated field is the string `"unknown"`.**
   Never use `null`, empty strings `""`, or `0`.
3. 🔴 **Never fabricate plausible fake fallbacks.**
   A value that looks plausible is worse than no value because it is trusted and deceives operators.
   Specifically forbidden:
   - **Dynamic `builtAt`:** Setting `builtAt` to the current timestamp on every request (`pos108-sell` incident).
   - **Static hardcoded `builtAt`:** Hardcoding a past date like `'2026-09-01T00:00:00Z'` (`108heros-web#16`, `pos108-admin`).
   - **Fake commit SHA:** Emitting `build: "sha-local"` on deployed environments.
   - **Guessed channel:** Inferring `channel` from `NODE_ENV`.
4. **`"unknown"` is only acceptable during local development.**
   In any deployed environment (`staging` or `release`), `"unknown"` indicates a broken build or
   deployment pipeline and is treated as a defect.

---

## 4. The version must survive the trip to the cluster

A number that is correct in the repo and wrong on the cluster is worse than no number,
because it is trusted.

- **Deploy through helm, never around it.** `kubectl set image` writes the Deployment and
  leaves the helm release believing something else; that is exactly how `tenant-admin`
  drifted 32 days (`pos108-admin#541`) and how `jh-api` drifted onto a package name from
  two renames ago (`jh-api#373`). Pass the image on every deploy —
  `helm upgrade <rel> <chart> --reuse-values --set image.tag=sha-<short>` — so the release
  is self-correcting.
- **Never write a floating tag into a values file.** `latest` and a bare chart
  `appVersion` are both tags this ecosystem does not publish; a values file that names one
  is a rollback waiting for someone to run `helm upgrade` for an unrelated reason
  (`108jobs-web#112`, `108heros-web#12`).
- **Verify a deploy by identity, not by liveness.** HTTP 200 can come from the old pod.
  Read the version back from §3, or read the image off the pod (not off the Deployment —
  `set image` can succeed while the pull fails and the old ReplicaSet keeps serving).

---

## 5. Schema is a version too

`pos108-core` reports `schemaVersion` — the highest applied migration — next to its build.
Keep that, and copy it into any service that owns a database. It is the only field in the
health payload that **cannot be faked by a stale pod**, which makes it the cheapest proof
that a migration actually ran, as opposed to an image merely rolling.

---

## 6. Compliance

A repo is compliant when all seven hold:

- [ ] the number follows §2 and lives in the language's own manifest
      (`Cargo.toml` / `package.json` / `pubspec.yaml`) — one place, not two
- [ ] the running app reports `version` / `build` / `builtAt` / `channel` per §3 (all four fields present, `"unknown"` if missing)
- [ ] a front end also reports the version of the backend it is bound to
- [ ] deploy goes through helm with the image passed explicitly (§4)
- [ ] **CI fails if the reported version does not match the manifest** — without this the
      field drifts and we are back to decoration
- [ ] **CI fails if runtime identity reports `"unknown"` on `staging` or `release` builds**
- [ ] **Tests assert real injection, not fallbacks.** Tests like `expect(info.build).toBeDefined()` or
      `toBeDefined()` on a date constructor against hardcoded fallbacks are tautologies and do not
      satisfy compliance. Tests must prove build metadata is passed into the binary/bundle.

### Current standing — survey updated 2026-09-02 (#17)

| repo | §2 number | §3 runtime surface | §4 helm deploy |
|---|---|---|---|
| `pos108-core` | ✗ `0.1.0` frozen | ✅ `/health/ready`, full payload | ✗ drift (~2.5 wks) |
| `pos108-terminal` | ✅ real, moves | ✅ update check | n/a (installed app) |
| `pos108-admin` | ✗ frozen | ✗ fake fallback (`builtAt` fixed to `2026-09-01T00:00:00Z`) | ✅ `#543` merged |
| `pos108-orders` | ✗ frozen | ✗ missing env, fallback tautology | ✗ drift (~1 mo) |
| `pos108-store` | ✗ `0.0.0` | ✗ static `version.json`, missing `VITE_APP_BUILD` | ✗ drift (~3 wks) |
| `pos108-sell` | ✗ frozen | ✗ fake fallback (`builtAt` computed on every request) | ✗ drift (~3 wks) |
| `park108` | ✗ frozen | ? not surveyed | ? |
| `jh-api` | grandfathered | ? not surveyed | ✅ |

`?` means **not yet surveyed** — not "compliant". The 2026-09-02 survey (#17) proved that
all four web frontends (`sell`, `admin`, `store`, `orders`) relied on fake fallbacks and
tautological assertions rather than true injection. Replacing those fallbacks with `"unknown"`
and requiring real injection in CI is tracked in each repo's issue.

---

## 7. What this document does not decide

- **When a release train is cut, and by whom.** MINOR moves on a decision, and nobody has
  written down whose decision it is.
- **Whether a shop can stay on an older build.** Today every shop moves together, because
  one shared bundle serves all of them (`pos108-admin#542`). Per-shop pinning is a design
  question, not a versioning one, and this standard does not assume either answer.
