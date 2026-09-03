# 108 Ting Ecosystem — K3s Deployment Standard (staging + production)

> **Status:** STANDARD — owner-decided 2026-06-19. This is the authoritative
> "where does each service deploy" reference for the ecosystem. Read-only planning
> doc — it does **not** change any service code, compose file, or CI yet; each
> rollout wave gets its own implementation slice + PR.
>
> **Supersedes** the root `deploy/` docker-compose stack *as the staging/production
> target* (compose stays for **local dev only** — see §10). Per-service deploy docs
> (e.g. Notification Helm chart, pos108 `docs/DEPLOYMENT.md`) remain the source of
> truth for that service's internals; this doc governs **placement + cluster
> conventions** across all of them.
>
> **Related:** [`PRODUCTION_READINESS_AUDIT.md`](PRODUCTION_READINESS_AUDIT.md) ·
> [`PRODUCTION_READINESS_BAR.md`](PRODUCTION_READINESS_BAR.md) ·
> [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`../ENGINEERING_CONSTITUTION.md`](../ENGINEERING_CONSTITUTION.md) ·
> root [`../deploy/README.md`](../deploy/README.md)

---

> ## ⚡ Architecture update — PROVEN nginx-edge model (2026-06-19)
>
> The design below originally assumed **K3s Traefik owns `:80/:443`** with wildcard TLS
> via RFC2136. **The Dell's reality differs and the proven path is simpler:**
> - The Dell already runs **system nginx on `:80/:443`** (certbot TLS) fronting live
>   sites (`api.108plaza.net` → `127.0.0.1:18000`, …). K3s has **no ingress controller**.
> - So K3s services are exposed as **NodePort**, and the existing nginx **reverse-proxies**
>   each hostname → `127.0.0.1:<nodePort>`, with **certbot** terminating TLS per host.
> - **Traefik + RFC2136 (Appendix A) are PARKED** — not used unless K3s ever owns the edge.
>
> **Proven:** Notification is live this way — `https://notify.staging.108plaza.net/health`
> → HTTP/2 200, via nginx → NodePort 30084 → pod → pg/redis. Per-service recipe =
> NodePort + nginx vhost (`deploy/k3s/nginx/`) + `certbot --nginx -d <host>`.
> Read §4 / §9 / Appendix A through this lens.

## 1. Decisions locked (owner, 2026-06-19)

| # | Decision | Value |
|---|----------|-------|
| **D1** | Cluster topology | **One K3s cluster, two environment namespaces** (`prod` / `staging`). No separate staging cluster. |
| **D2** | Cluster node | **Single node** — the **Dell R645 (Linux)** as combined master+worker, running **both** `prod` and `staging` namespaces. Chosen for the most stable network/port config on one Linux OS. |
| **D3** | Mac mini `.68` | **Stays OUT of the K3s cluster.** Runs external databases / legacy systems only — not joined as a node (keeps cluster networking simple). |
| **D4** | Cluster scope | K3s hosts **central/cloud services only.** Offline-first POS **branch** nodes and **IoT edge** gateways deploy **at-site**, outside the cluster. |

**Derived from the locks (rationale in the sections noted):**
- **D5** — Stateful data is external to the cluster: **prod** databases live on the Mac mini `.68` (existing `postgres@16` + Redis); **staging** databases run **in-cluster** as small StatefulSets so staging is self-contained and disposable. (§7)
- **D6** — Single TLS edge: the Dell's **existing system nginx** (certbot) holds `:80/:443`; K3s services are **NodePort** and nginx reverse-proxies each hostname → `127.0.0.1:<nodePort>`. (Originally planned as K3s Traefik owning the ports — superseded; see the banner above + §9.)

---

## 2. Why these locks (the shape in one paragraph)

A single beefy Linux node (Dell R645) runs one K3s cluster. Inside it, two
namespaces — `prod` and `staging` — hold the same set of **stateless** central
services, isolated by NetworkPolicy + ResourceQuota and addressed by distinct
hostnames. **State lives outside the cluster**: production databases on the Mac
mini `.68` (reached over the LAN), staging databases as throwaway in-cluster
StatefulSets. **Offline-first POS** branch terminals and **IoT edge** gateways are
deliberately *not* in the cluster — they run next to the hardware / the cash
drawer and survive a WAN outage. The cluster is therefore the **central plane**
(auth, the POS cloud aggregator, the async consumers, analytics, public product
backends), never the edge.

```
                          ┌──────────────────────────────────────────────────────┐
                          │  Dell R645 (Linux)  —  single-node K3s cluster         │
                          │                                                        │
   Internet / LAN ──:443──┤  Traefik ingress (host :80/:443, host-based routing)   │
                          │     │                                                  │
                          │     ├── namespace: prod      ┌── identity-api          │
                          │     │   (real domains)       ├── pos108 (cloud agg.)   │
                          │     │                         ├── accountzing           │
                          │     │                         ├── payment               │
                          │     │                         ├── notification (+NATS)  │
                          │     │                         ├── media / creator / …   │
                          │     │                         └── data-platform (async) │
                          │     │                                                   │
                          │     └── namespace: staging   └── same set + in-cluster  │
                          │         (staging.* domains)      PG/Redis/NATS          │
                          │                                                        │
                          │  infra ns: kube-system(Traefik) · cert-manager ·        │
                          │            sealed-secrets · monitoring                  │
                          └───────────────┬────────────────────────────────────────┘
                                          │ LAN (prod DB connections only)
                          ┌───────────────┴───────────────┐
                          │  Mac mini .68 (NOT a K3s node) │
                          │  postgres@16 · Redis · legacy  │   ← prod state lives here
                          └────────────────────────────────┘

   ── OUTSIDE the cluster (deploy at-site, offline-capable) ─────────────────────
     POS branch terminals (pos108 APP_ENVIRONMENT=branch)  →  each store
     IoT edge gateways (Smart-Farm pumps/valves, Smart-Home Home Assistant)  →  each site
```

---

## 3. Namespace & naming conventions

**Two workload namespaces** (D1):

| Namespace | Purpose | Hostnames | DB source |
|-----------|---------|-----------|-----------|
| `prod` | Production central plane | `<svc>.108plaza.net` (§10) | Mac mini `.68` (external) |
| `staging` | Pre-prod / integration | `<svc>.staging.108plaza.net` (§10) | in-cluster StatefulSets |

**Shared platform namespaces** (standard infra, not "the two app namespaces"):
`kube-system` (Traefik, CoreDNS, metrics-server — K3s built-ins) · `cert-manager`
· `sealed-secrets` · `monitoring` (Prometheus + Grafana).

**Isolation between `prod` and `staging` on the shared node:**
- `ResourceQuota` + `LimitRange` per namespace — staging cannot starve prod of CPU/mem.
- `NetworkPolicy` default-deny cross-namespace — a staging pod cannot reach prod DB or prod pods.
- `PriorityClass`: `prod` workloads > `staging` so prod wins under contention.

**Naming:** one Helm release per service per namespace; resource names
`<service>` within the namespace (namespace already disambiguates env). Images
tagged `ghcr.io/108-plaza/<service>:<git-sha>` — **no `:latest` in `prod`** (pin to
an immutable tag for reproducible rollbacks).

---

## 4. Shared cluster services (the platform layer — install once)

| Concern | Standard | Notes |
|---------|----------|-------|
| **Ingress** | **Existing system nginx** (edge) → K3s **NodePort** | The Dell's nginx already owns `:80/:443` for live sites; add a vhost per service proxying `host → 127.0.0.1:<nodePort>` (templates in `deploy/k3s/nginx/`). K3s has no ingress controller. (Traefik-in-K3s = parked alternative, Appendix A.) |
| **TLS** | **`certbot --nginx`** per host (HTTP-01) at the edge | nginx owns `:80` → certbot issues per-host Let's Encrypt certs automatically (same as the existing `108plaza.net` vhosts), no wildcard/DNS-01 needed. HSTS terminated at nginx. (RFC2136 wildcard via Traefik = parked, Appendix A.) |
| **Secrets** | **Sealed Secrets** (Bitnami) | Encrypted secrets committed to git → GitOps-safe on a single node. Path to **External Secrets Operator + Vault/KMS** later (the audit's open "no central secret manager" item). **Never** plain `Secret` YAML in git. |
| **Registry** | **GHCR** `ghcr.io/108-plaza/*` | Already on GitHub org + self-hosted runners. Images are **built + pushed only from the `deploy/staging` branch** — never from `main` (§4a). Cluster pulls with an `imagePullSecret`. |
| **CI → deploy** | Advance `deploy/staging` → image-build CI publishes the tag → `helm upgrade --install` to that tag | Image build is **branch-triggered** (`deploy/staging`), not per-merge-to-`main` (§4a). The `helm upgrade` step is still a separate, owner-gated action today; **Argo CD / Flux** is the Phase-2 pull-based-GitOps upgrade once >~3 services are live. |
| **Observability** | **Prometheus + Grafana** in `monitoring` ns | Scrape `/metrics` (pos108 `#310`, Notification, Payment, Data already expose it). Alerting: outbox backlog, dead-letter, consumer lag, ledger imbalance. |
| **Message broker** | **NATS** per workload namespace (Deployment/StatefulSet) | Notification needs it; replaces the compose `infra.yml` NATS. |
| **DNS to external DB** | `ExternalName` Service / `Endpoints` object in `prod` → Mac mini `.68` | Manifests reference a stable in-cluster name (e.g. `pg-central.prod.svc`) instead of a raw IP. |

**HA reality (state honestly):** one node = **no control-plane HA**, and prod
state on one Mac mini = single box too. Resilience here is **backup/restore +
fast redeploy**, not live failover. Acceptable for current scale; revisit
(add a 2nd K3s node / managed PG) when uptime SLA demands it.

---

## 4a. Image build & publish — the `deploy/staging` trigger model (owner-decided 2026-06-25)

**`main` never builds an image.** Pushing/merging to `main` runs tests, lint, and the
rest of CI — but **no Docker build and no GHCR push**. Image builds are slow and `main`
churns constantly; keeping builds off `main` keeps the dev loop fast and stops the
registry filling with artifacts nobody deploys.

**`deploy/staging` is the build branch.** Every image-building repo carries one
long-lived **`deploy/staging`** branch, and the image-build CI fires **only on push to
`deploy/staging`** (`workflow_dispatch` stays for manual one-offs). Two wiring patterns,
depending on how the repo is shaped today:

| Repo shape | Files | Trigger change |
|------------|-------|----------------|
| **Dedicated image workflow** | `release-image.yml`, `docker.yml` | `on: { push: { branches: [deploy/staging] }, workflow_dispatch: {} }` |
| **Image job inside `ci.yml`** | Notification `image` (the only repo that publishes from `ci.yml`) | add `deploy/staging` to `on.push.branches`; gate the image job `if: github.ref == 'refs/heads/deploy/staging'`. The rest of `ci.yml` (test/lint) still runs on `main` + PRs. |

**Flow to cut a staging image:**

1. Land the change on `main` the normal way (PR → review → CI green).
2. Advance the build branch — `git push origin main:deploy/staging` (fast-forward), or
   merge `main → deploy/staging`.
3. The push to `deploy/staging` triggers the image-build workflow → publishes
   `ghcr.io/108-plaza/<svc>:sha-<short>` (+ `:staging`).
4. Deploy through Helm (never around it): run `helm upgrade <rel> "$CHART_PATH" -n <ns> --reset-then-reuse-values --set image.repository="ghcr.io/108-plaza/<svc>" --set image.tag="sha-<short>" --wait --timeout 3m`, followed by `kubectl rollout status` and verifying `readyReplicas >= 1` (see [HELM_DEPLOY_PORTING_GUIDE.md](file:///Users/yuth/Documents/108-Ting-Ecosystem/HELM_DEPLOY_PORTING_GUIDE.md)). Passing both repo and tag on every run ensures the release self-corrects and prevents drift.

**Uniform across the ecosystem.** Same branch name (`deploy/staging`), same trigger, in
every image-building repo: **pos108, pos108-admin, pos108-orders, Payment-Platform,
tix-tox-clone, imageprocessing, Identity-Platform, Notification-Platform,
AccountZing-Platform**.

**Supersedes the dev-phase stopgap.** This replaces the 2026-06-23 `workflow_dispatch`-only
setting (image builds disabled on every push/PR). Builds are no longer purely manual —
they fire automatically when you **intentionally advance `deploy/staging`**, while `main`
stays build-free.

**Prod** image promotion (retag/promote a proven `deploy/staging` sha, or an analogous
`deploy/prod` branch) is a **separate decision — out of scope here**; `deploy/staging` is
the only build branch today.

---

## 5. Per-service placement matrix — "ตัวไหนวางตรงไหน"

Legend — **IaC**: ✅ ready · 🟡 partial (Dockerfile/compose, needs k8s/Helm) · ❌ none (build first).

### 5a. Central services → **K3s** (`prod` + `staging` namespaces)

| Service | Repo | Namespace(s) | Data store | IaC | Placement notes |
|---------|------|--------------|-----------|-----|-----------------|
| **identity-api** | Identity-Platform | prod, staging | Mac mini PG `identity` (prod) / in-cluster (staging) | 🟡 compose+prod | Auth edge, Ed25519 JWT issuer. Boot order #1. Prod secrets now required (audit #2 cleared). Model Helm on Notification chart. |
| **pos108 (cloud)** | pos108 | prod, staging | Mac mini PG `pos108` + Redis | 🟡 Dockerfile fixed (#302) | **CLOUD aggregator only** (`APP_ENVIRONMENT=cloud`: inbox applier + conflict resolver). **Branch nodes do NOT go here** (§5c). Probes `/health/live`,`/health/ready`; grace 35s. |
| **accountzing** | AccountZing-Platform | prod, staging | Mac mini PG `accountzing` | 🟡 **Dockerfile + prod compose ✅ (#12)** | Dockerfile + `docker-compose.prod.yml` landed (#12, 2026-06-17); prod values drafted (`examples/values.accountzing.prod.yaml`). Remaining: seal secrets + deploy. Internal-only (ClusterIP). Auth + scheduler wired (#7/#8). Ledger consumer of pos108 outbox. |
| **payment** | Payment-Platform/Gateway | prod, staging | Mac mini PG `payment` | 🟡 compose+prod | Consumer. Audit follow-ups merged (metrics+outbox+Docker #9). Single-PSP (SCB) today. |
| **notification** | Notification-Platform | prod, staging | staging in-cluster PG/Redis (prod → Mac mini) | ✅ **LIVE in staging** | **The proven reference.** Deployed via shared chart (NodePort 30084) → nginx vhost + certbot → `https://notify.staging.108plaza.net` 200. Recipe for every other service. |
| **media** | Media-Platform | prod, staging | Mac mini PG / object storage | 🟡 artifacts added (#3) | 9 services. Needs k8s manifests; CI now runs DB-adapter tests (#9). |
| **creator** | Creator-Platform | prod, staging | PG | ✅ has k8s (`k8s/*.yaml`) | Adapt existing manifests to namespaces + Traefik + Sealed Secrets. |
| **delivery** | Logistics-Platform | prod, staging | PG | 🟡 compose | Needs k8s manifests. |
| **data-platform (Stratum)** | Data-Platform | prod, staging *(async, lower priority)* | own: Redpanda + MinIO + PG | 🟡 compose | **Async analytics — explicitly OUT of the POS runtime critical path** (owner-locked offline-first baseline). Money-Gold path still gated by the reconciliation gate (audit #5) — do not ship money analytics until built. |

### 5b. Public-product backends → **K3s candidate** (respect each repo's own launch decision)

| Service | Repo | Namespace(s) | IaC | Placement notes |
|---------|------|--------------|-----|-----------------|
| **BipByte server (108Zing)** | tix-tox-clone | prod, staging *(later wave)* | 🟡 compose staging/prod + design | **60 services.** Its own approved decision (D-PD1 = Option A) is **compose-on-VM for first launch, k8s post-launch**. Honor that: onboard to K3s as the *post-launch packaging exercise* (codebase is already k8s-ready). Heavy for one node — capacity-check before prod. Money-path PG split (D-PD3) carries over. |
| **BipByte realtime-edge (E5)** | tixtox-realtime-edge | prod, staging | ❌ not deployed | Phoenix edge (separate design). K3s or a dedicated edge VM; needed for realtime fan-out. |
| **BipByte engines (image-proc)** | imageprocessing | — | C++/GPU | **Out of general K3s** (GPU placement is `tixtox-engines` scope). Only its Redis/ingest contract touches the stack. Place on a GPU host, not the Dell general pool, unless the Dell has a suitable GPU. |
| **Frontends** (pos108-admin/orders/pos, slot-front; tixtox-web, tixtox-admin-web) | various | prod, staging | mixed | **Next.js/SSR** → Deployments behind Traefik. **Flutter web builds** (static `build/web`) → static-serve via a tiny nginx pod **or** object storage + CDN (cheaper, offloads the node). |

### 5c. **OUTSIDE the cluster** — deploy at-site (D3/D4)

| Thing | Where it runs | Why not in K3s |
|-------|---------------|----------------|
| **pos108 branch terminals** | At each store (`APP_ENVIRONMENT=branch`: push/pull/heartbeat/resolver) | **Offline-first, owner-locked.** Must keep selling during a WAN outage; talks to the cloud node via `APP_SYNC__CLOUD_BASE_URL` + `APP_SYNC__BRANCH_API_KEY`. Deploy: single binary / compose at-site. |
| **IoT edge — Smart-Farm gateway** | At each farm | Drives real pumps/valves; must run next to hardware and survive network loss. (Its *cloud backend* portion may join K3s later; the **gateway** does not.) |
| **IoT edge — Smart-Home** | At each home | Home Assistant at-home, near devices. |
| **Production Postgres@16 + Redis** | **Mac mini `.68`** | D3 — external data services, kept off the cluster on purpose. |
| **Legacy systems** | Mac mini `.68` | As-is. |

---

## 6. Environment standard — what differs `prod` vs `staging`

| Aspect | `prod` | `staging` |
|--------|--------|-----------|
| Databases | Mac mini `.68` (`postgres@16`, real DBs) | in-cluster StatefulSet PG/Redis/NATS (disposable) |
| Hostnames | real domains, public TLS (Let's Encrypt prod issuer) | `staging.*`, staging issuer (or self-signed) |
| Secrets | Sealed Secrets (prod keys) | Sealed Secrets (staging keys) — **never reuse prod secrets** |
| Replicas / HPA | HPA on (e.g. Notification min 3 / max 10) | low fixed replicas (1) to save the node |
| Resource quota | majority share + higher `PriorityClass` | capped quota, lower priority |
| `pos108` env | `APP_ENVIRONMENT=cloud` | `APP_ENVIRONMENT=cloud` (staging data) |
| Image tag | pinned `:<git-sha>` (no `:latest`) | `:<git-sha>` or `:staging` |
| Bootstrap admin | `false` | `true` only for first seed, then off |

---

## 7. Data & stateful policy (D5)

- **Prod state is external** (Mac mini `.68`). Pods connect over the LAN via a
  stable in-cluster `ExternalName`/`Endpoints` Service. One PG instance with a
  database + least-privilege role per service (the existing
  `deploy/postgres-init/01-init-databases.sql` pattern carries over).
  - **Money-path split** (carry over BipByte D-PD3 thinking): keep
    finance-critical DBs (AccountZing ledger, Payment, BipByte wallet/ledger/gift)
    isolated — separate instance or at least separate role/backup policy from the
    general shared DB.
- **Staging state is in-cluster** and disposable — small `postgres:16-alpine` /
  `redis:7-alpine` / `nats:2-alpine` StatefulSets (mirrors today's `infra.yml`),
  with modest PVCs on the Dell's local storage (K3s `local-path` StorageClass).
- **Migrations** run at boot (`sqlx::migrate!`); a failed migration aborts startup
  and the old pod keeps serving until the new one is `Ready` → safe rollback.
  Additive-only migration discipline already enforced in pos108.
- **Backups** are the resilience story (no HA): scheduled `pg_dump`/PITR on the Mac
  mini prod DBs + a **rehearsed restore drill** (open audit item for BipByte;
  make it the ecosystem standard). Staging needs none (disposable).

---

## 8. Rollout waves (gated by IaC readiness, not by wishful order)

> Onboard in dependency order, and only when a service actually has a deployable
> image + manifests. Prove each wave in `staging` before `prod`.

- **Wave 0 — Platform foundation (one-time):** K3s **already runs on the Dell tower**
  → *configure, not install* (full checklist in [`K3S_ONBOARDING_PLAYBOOK.md`](K3S_ONBOARDING_PLAYBOOK.md) §1).
  Configure **Traefik ACME DNS-01 via RFC2136** (self-hosted BIND) for the wildcard certs (Appendix A);
  install Sealed Secrets + Prometheus+Grafana (cert-manager optional — internal
  certs only); create `prod`/`staging` namespaces with quotas/NetworkPolicy/
  PriorityClass; wire the GHCR pull secret; create the `ExternalName` to Mac mini
  PG; stand up staging in-cluster PG/Redis/NATS.
- **Wave 1 — Notification** (✅ Helm ready): the reference rollout. Validates
  ingress, TLS, secrets, metrics, NATS end-to-end. Locks the per-service template.
- **Wave 2 — Core auth + POS path:** **Identity** → **pos108 (cloud)** → **Payment**.
  Author Helm/manifests modeled on Wave 1. This is the boot-order spine
  (Identity → pos108 → consumers).
- **Wave 3 — Remaining consumers/services:** **AccountZing** (🔴 build
  Dockerfile + manifests **first**), **Media**, **Creator** (adapt existing k8s),
  **Logistics/delivery**.
- **Wave 4 — Async + public + frontends:** **Data-Platform** (after reconciliation
  gate; async, never POS-critical), **BipByte** (align with its own
  compose-first launch decision; k8s as post-launch), **frontends/CDN**, IoT
  **cloud** backends. Edge stays at-site.

---

## 9. Network & port plan (single Linux host — proven model)

The Dell's **existing system nginx owns `:80/:443`** (certbot TLS) and fronts the live
sites. K3s has no ingress controller, so K3s services join the **same pattern as just
another nginx backend**:

- Each K3s service is a **NodePort** (e.g. Notification `30084`). nginx adds a vhost
  `server_name <host>; proxy_pass http://127.0.0.1:<nodePort>;` and **`certbot --nginx`**
  issues the per-host cert (HTTP-01, since nginx owns `:80`). Templates: `deploy/k3s/nginx/`.
- **External traffic**: `notify.staging.108plaza.net` (DNS → the Dell `103.27.202.40`)
  → nginx `:443` (TLS) → `127.0.0.1:30084` → kube-proxy → pod `:8080`. The DNS
  `*.staging` wildcard means new staging hosts need no DNS edit; each still gets its
  own nginx vhost + cert.
- **Service-to-service** stays in-cluster via ClusterIP DNS (`<svc>.<ns>.svc:80`,
  `pg-central.<ns>.svc:5432`, `redis-central.<ns>.svc:6379`, `nats.<ns>.svc:4222`).
  No host ports for east-west traffic.
- **Pods → prod DB**: pods dial `pg-central.prod.svc` (Endpoints → Mac mini `.68`).
  NetworkPolicy must allow **intra-namespace on ALL ports** — the original 8080-only
  ingress rule blocked app→`postgres:5432` (fixed in `03-networkpolicy.yaml`, 2026-06-19).
- **NodePort exposure**: NodePorts bind `0.0.0.0` → also reachable on the Dell's public
  IP. Lock with a firewall (or `kube-proxy --nodeport-addresses=127.0.0.1`) so only the
  local nginx reaches them.

> **Parked alternative:** if K3s ever takes the edge, install Traefik on `:80/:443`,
> route by Host with ClusterIP services + IngressRoutes + RFC2136 wildcard TLS
> (Appendix A). Not used today — nginx already owns the edge.

---

## 10. Subdomain & DNS naming standard (owner-decided 2026-06-19)

**Locked:** single apex **`108plaza.net`**, **flat** per-service subdomains, environment
encoded as a `staging` zone. (Chosen over a brand-split apex; the existing
`pos108.com` / `108zing.108plaza.net` hosts converge onto this — see migration notes.)

### Convention

| Environment | Pattern | → Namespace | Example |
|-------------|---------|-------------|---------|
| **prod** | `<service>.108plaza.net` | `prod` | `pos.108plaza.net` |
| **staging** | `<service>.staging.108plaza.net` | `staging` | `pos.staging.108plaza.net` |

- **Internal service-to-service traffic does NOT get a subdomain** — it uses cluster
  DNS `<service>.<namespace>.svc.cluster.local` (§9). Subdomains are *only* for
  traffic entering through Traefik from outside the cluster.
- **`<service>` slug** = short, lowercase, **single label** (no extra dots),
  identical across prod/staging — only the `staging` zone differs. One slug per
  service, owned in this table.

### TLS — two wildcards cover the whole fleet

- `*.108plaza.net` → prod · `*.staging.108plaza.net` → staging. Adding a new service
  needs **no new certificate**.
- Wildcards require a **DNS-01** solver (HTTP-01 cannot issue wildcards). `108plaza.net`
  DNS is **self-hosted BIND** (`ns1`/`ns2.108jobs.com`) — name.com is only the registrar —
  so issuance is done by **Traefik's built-in ACME** via the **RFC2136** provider (a
  TSIG-signed dynamic update sets the `_acme-challenge` TXT straight on your BIND).
  Traefik requests + auto-renews. Concrete manifests + BIND setup in **Appendix A** /
  `deploy/k3s/`.

### DNS records

- `*.108plaza.net` + `*.staging.108plaza.net` → **A record → Dell R645 public IP**
  (Traefik). Everything else is Traefik host-routing on `:443`. (Pin explicit
  records later if you prefer them over wildcard A records.)

### Public subdomain grid — who actually gets a name

> Only **edge** services get a subdomain. Internal consumers / the money ledger
> stay ClusterIP-only.

**Public (nginx vhost + subdomain) — prod host shown; staging = same slug under `.staging.108plaza.net`:**

| Service | prod host | Notes |
|---------|-----------|-------|
| Identity / auth | `auth.108plaza.net` | login, JWKS, OAuth — align `JWT_ISSUER` to this |
| pos108 cloud | `pos.108plaza.net` | branch-sync + admin API target |
| Notification | `notify.108plaza.net` | admin API + inbound delivery callbacks |
| Payment | `pay.108plaza.net` | inbound PSP (SCB) webhooks |
| Media | `media.108plaza.net` | asset serving (use `cdn.` if CDN-fronted) |
| Data-Platform (Stratum) | `stratum.108plaza.net` | already real; IP-allowlist / VPN (internal analytics) |
| Smart-Farm cloud | `smartfarm.108plaza.net` | already real |
| Creator | `creator.108plaza.net` | |
| Logistics | `delivery.108plaza.net` | driver / tracking + webhooks |
| 108Zing gateway | `bipbyte.108plaza.net` | public product API (was `api.108zing.108plaza.net`) |
| 108Zing realtime | `bipbyte-ws.108plaza.net` | Phoenix edge WS (was `ws.108zing.108plaza.net`) |
| POS app (frontend) | `app.108plaza.net` | was `app.pos108.com` |
| POS admin | `admin.108plaza.net` | |
| POS orders | `orders.108plaza.net` | |
| POS slot | `slot.108plaza.net` | |
| 108Zing web | `bipbyte-web.108plaza.net` | |
| 108Zing admin | `bipbyte-admin.108plaza.net` | |

**Internal-only (NO subdomain — ClusterIP, reached service-to-service):**
- **AccountZing** (money ledger) — deliberately **not** public; reached at
  `accountzing.<ns>.svc`. Keeps the ledger off the internet (defense-in-depth on
  top of its new auth #8).
- **BipByte internal services** (user / feed / wallet / ledger / gift / messaging /
  … ≈ 50 of the 60) — behind the `zing` gateway only.
- Any pure consumer with no inbound-HTTP need.

### Migration notes (existing real hosts → this standard)

- `api.108zing.108plaza.net`, `ws.108zing.108plaza.net`, `staging.108zing.108plaza.net`
  → re-point to the **flat** `bipbyte.*` / `bipbyte-ws.*` names. The nested `108zing.`
  label would need its *own* extra wildcard pair, which the single-apex/two-wildcard
  plan avoids. **If** 108Zing must keep the `108zing.108plaza.net` brand, that
  sub-zone is the *one allowed exception* and owns its own
  `*.108zing.108plaza.net` + `*.staging.108zing.108plaza.net` certs.
- `app.pos108.com` → canonical becomes `app.108plaza.net`; keep `pos108.com` only as
  an optional public **vanity 301-redirect** to the canonical if the merchant-facing
  brand matters.
- `mcp.108plaza.com` (`.com`) → leave as-is (out of cluster, MCP infra); fold to
  `.net` later if desired. Not part of this grid.

---

## 11. What this standard does NOT change

- **Local dev stays on docker-compose.** Root `deploy/` (base + `infra.yml`) and
  each repo's `docker compose up` remain the laptop/dev workflow. K3s is the
  **staging + production** target only.
- **No service code / CI / compose edits in this doc** — placement only. Each wave
  is a separate slice + PR (Dockerfile fixes, Helm charts, manifests).
- **Per-service operational docs win on internals** — Notification's chart,
  pos108's `DEPLOYMENT.md`/`RUNBOOK.md`, etc., remain authoritative for *how each
  service runs*; this doc governs *where it lands*.

---

## 12. Open items / follow-ups (track before prod traffic)

1. **BIND TSIG key + `*.staging` A record** — DNS = self-hosted BIND. Remaining:
   `tsig-keygen` + named.conf `update-policy` on `_acme-challenge` (zone goes dynamic →
   `rndc freeze`/`thaw` for manual edits); fix the malformed legacy wildcards; add
   `*.staging IN A <DELL-IP>`. ACME issues via RFC2136 even before A records exist; test
   on LE **staging** CA first. Paste-ready in `deploy/k3s/README.md`.
2. **AccountZing deploy artifacts** 🟡 — Dockerfile + prod compose landed (#12, 2026-06-17); prod
   values drafted. Remaining = manifests/values wiring + secrets + deploy, not the Dockerfile.
3. **Central secret manager** — Sealed Secrets is the start; decide if/when to
   move to External Secrets + Vault/KMS (audit cross-cutting #6).
4. **GitOps tooling** — Argo CD vs Flux for Phase-2 (after Wave 1–2 prove the
   `helm upgrade`-from-runner baseline).
5. **Backup/restore drill** — make the BipByte "rehearsed restore" an
   ecosystem-wide prod-readiness gate (Mac mini PG).
6. **Node capacity** — size BipByte's 60-service stack against the single Dell
   before committing it to `prod`; a 2nd node may be needed at that point.
7. **GPU placement** — decide host for tixtox-engines (image-proc) if the Dell has
   no suitable GPU.

---

## Appendix A — Traefik ACME via RFC2136 (PARKED ALTERNATIVE — not the live setup)

> 🅿️ **PARKED.** The live setup terminates TLS at the **existing system nginx** with
> **`certbot --nginx`** per host (§9) — Traefik is not installed in K3s. Keep this
> appendix only for the future case where K3s takes the `:80/:443` edge itself; then
> RFC2136 wildcard issuance below applies. Everything below is the *alternative*, not
> the current standard.

> **Note:** `108plaza.net` DNS is **self-hosted BIND** (`ns1`/`ns2.108jobs.com`),
> not name.com-managed — name.com is only the registrar. So the lego `namedotcom`
> provider can't issue (it writes to name.com's non-authoritative DNS). Issuance uses
> **RFC2136** (TSIG dynamic update) straight to your BIND. **Live manifests + the
> paste-ready BIND/TSIG/zone setup are in [`../deploy/k3s/`](../deploy/k3s/) (README
> "BIND / RFC2136 setup" + `04-traefik-acme.yaml` + `04-acme-rfc2136-secret.template.yaml`).**

**0. BIND side** — `tsig-keygen -a hmac-sha256 acme-update`; add the key + an
`update-policy` granting it `_acme-challenge.108plaza.net` + `_acme-challenge.staging.108plaza.net`
TXT; the zone becomes dynamic (edit later via `rndc freeze`/`thaw`). Add
`*.staging IN A <DELL-IP>` (+ fix the malformed legacy wildcards). Full block in the
deploy/k3s README.

**1. TSIG Secret** (`kube-system/acme-rfc2136`) — `RFC2136_NAMESERVER` (ns1:53),
`RFC2136_TSIG_KEY` (`acme-update`), `RFC2136_TSIG_ALGORITHM` (`hmac-sha256.`),
`RFC2136_TSIG_SECRET` (from tsig-keygen). Seal it.

**2. Traefik customisation** (`HelmChartConfig`) — same as `deploy/k3s/04-traefik-acme.yaml`:
the `le` resolver with `--…dnschallenge.provider=rfc2136`, the `RFC2136_*` env from the
secret, and persistence for `acme.json`. Issues `*.staging.108plaza.net` now
(+ `*.108plaza.net` when prod moves to the Dell).

**3. Per-service route** (Traefik `IngressRoute`) — unchanged. The `tls.domains` block is what
asks `le` to mint the wildcard; every other service just reuses `certResolver: le`
with the same wildcard SAN, so no new cert per service. Example — pos108, prod:
```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: pos108
  namespace: prod
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`pos.108plaza.net`)
      kind: Rule
      services:
        - name: pos108        # ClusterIP Service in ns prod, port 80 → container 8080
          port: 80
  tls:
    certResolver: le
    domains:
      - main: "108plaza.net"
        sans: ["*.108plaza.net"]
```
Staging is identical with `namespace: staging`, `Host(\`pos.staging.108plaza.net\`)`,
and `domains: [{ main: "staging.108plaza.net", sans: ["*.staging.108plaza.net"] }]`.

**4. Verify:** `kubectl -n kube-system logs deploy/traefik | grep -i acme` shows the
challenge; `curl -vI https://pos.staging.108plaza.net` returns a valid LE cert.
Switch the CA server arg from staging→prod once green.

> **Alternative (only if you want Certificate CRDs / non-Traefik consumers):**
> cert-manager has a **native `rfc2136` solver** — same TSIG key, issued as a
> `Certificate` resource. More moving parts than Traefik-managed; not needed for the
> single-node baseline.

---

*Authored 2026-06-19 as the ecosystem deployment standard. Update this file when a
wave completes or a locked decision changes.*
