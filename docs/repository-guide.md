# Open ATProto Extraction Repositories

This guide describes each public repository created as part of the **Stygian Tech Shared Extraction Plan**. Together they implement a three-tier model:

1. **Build-your-own packages** — reusable Swift/npm libraries
2. **Self-hosted reference services** — runnable gateways and backends you can deploy yourself
3. **Hosted SaaS** — L@tr.link and The Social Wire continue to use these packages/services in production

All repos live under the [Stygian-Tech](https://github.com/Stygian-Tech) GitHub organization.

---

## Architecture overview

```mermaid
flowchart TB
  subgraph foundations [Foundation packages]
    core[atproto-primitives]
    atproto[atproto-auth-kit]
    gateway[gateway-trust-kit]
    content[federation-content-kit]
    mcp[mcp-server-kit]
    sync[offline-sync-kit]
  end

  subgraph latr [L@tr domain]
    kit[latr-kit]
    npm[latr-packages]
    ref[latr-reference]
  end

  subgraph social [Social Wire domain]
    avkit[social-wire-appview-kit]
    swpkg[social-wire-packages]
    swref[social-wire-reference]
  end

  subgraph meta [Deployment]
    deploy[reference-deploy]
  end

  PDS[(Viewer PDS)]

  core --> atproto
  core --> gateway
  core --> content
  core --> mcp
  core --> sync
  atproto --> gateway

  kit --> ref
  npm --> ref
  ref --> PDS

  avkit --> swref
  swpkg --> swref
  swref --> PDS

  ref --> deploy
  swref --> deploy
```

---

## Foundation packages

These repos hold **generic ATProto infrastructure** with no product-specific UI or Social Wire/L@tr business rules. Other packages and services depend on them; they must not depend on product repos.

### [atproto-primitives](https://github.com/Stygian-Tech/atproto-primitives)

**Purpose:** Lowest-level shared primitives for all Stygian Swift packages.

**Contains:**
- `ATURI` — parse and validate `at://` record URIs
- `DID` — normalize and validate DIDs
- `NSID` — collection identifier parsing
- `Base64URL` — encoding helpers used by auth and trust code
- `StygianError` — shared error types

**Used by:** Every other Swift package in this extraction.

**Does not contain:** OAuth, HTTP servers, RSS, or L@tr/Social Wire record logic.

---

### [atproto-auth-kit](https://github.com/Stygian-Tech/atproto-auth-kit)

**Purpose:** ATProto client and auth foundations — the layer above `atproto-primitives` for talking to PDSes and verifying OAuth/DPoP.

**Contains (initial extraction):**
- `DPoPHtu` — RFC 9449 `htu` construction (query stripped, respects `X-Forwarded-Proto` behind Fly TLS)
- `ForwardedHTTP` — infer external scheme/host from proxy headers
- `OAuthScopes` — repo collection scope string builders

**Planned growth:** PDS/XRPC clients, JWT verification policy, upstream DPoP helpers, Jetstream/firehose primitives.

**Used by:** Gateway reference services, future `gateway-trust-kit` consumers, any Swift app doing ATProto OAuth.

**Depends on:** `atproto-primitives`, Hummingbird, NIO (for request types in DPoP helpers).

---

### [gateway-trust-kit](https://github.com/Stygian-Tech/gateway-trust-kit)

**Purpose:** Reusable **HTTP gateway** building blocks — CORS, internal service trust, proxy helpers — without product DTOs.

**Contains (initial extraction):**
- `GatewayInternalTrust` — HMAC signing/verification for Gateway → AppView internal requests (path-only signing to survive query re-encoding)

**Planned growth:** CORS policy, OAuth metadata route helpers, streaming authenticated proxy utilities.

**Used by:** `social-wire-reference` gateway, future extracted Social Wire gateway code.

**Depends on:** `atproto-primitives`, `atproto-auth-kit`, swift-crypto.

---

### [federation-content-kit](https://github.com/Stygian-Tech/federation-content-kit)

**Purpose:** **Content ingestion and normalization** — RSS, Jetstream, standard.site rendering, public URL helpers.

**Contains (scaffold targets):**
- `RssFeedKit` — RSS/Atom parsing, feed identity, stable item keys
- `JetstreamFirehoseKit` — WebSocket firehose subscriber protocol
- `StandardSiteRenderKit` — standard.site / Offprint field extraction
- `PublicResourceURLKit` — HTTPS URL normalization for federation-safe fetches

**Used by:** AppView worker, publication discovery, thumbnail/avatar normalization.

**Depends on:** `atproto-primitives`.

---

### [mcp-server-kit](https://github.com/Stygian-Tech/mcp-server-kit)

**Purpose:** **Scaffold** for a future Model Context Protocol server layer (tool registry, schema validation, transport). Part of the long-term Stygian productivity suite / MyContextProtocol integration.

**Status:** Placeholder package only — no production dependency yet.

**Depends on:** `atproto-primitives`.

---

### [offline-sync-kit](https://github.com/Stygian-Tech/offline-sync-kit)

**Purpose:** **Scaffold** for offline sync primitives — mutation queues, event cursors, reconciliation — intended for future Workspace/Tasks products, not L@tr or Social Wire today.

**Status:** Placeholder package only.

**Depends on:** `atproto-primitives`.

---

## L@tr domain packages

Everything related to **`com.latr.saved.*`** read-later records and the L@tr gateway HTTP API.

### [latr-kit](https://github.com/Stygian-Tech/latr-kit)

**Purpose:** **Authoritative Swift library** for L@tr on-protocol semantics. This is the server-side source of truth for how saves work on a user's PDS.

**Contains:**
- `SavedLibrary` — save URL, save native subject, archive, unsave, list workflows
- `RecordKey` — deterministic base32(SHA-256) rkeys for external wrappers and item edges
- `URLNormalizer` — canonical HTTPS URL normalization for deduplication
- `OpenGraphMerger` — merge OG preview fields onto records
- Lexicon-aligned models for `com.latr.saved.external` and `com.latr.saved.item`
- `RepositoryClient` protocol + test doubles

**Used by:** `latr-reference` (gateway), historically `latr-link/services/latr-gateway`.

**Contract:** Golden-vector tested; must stay aligned with `latr-packages` TypeScript record keys.

---

### [latr-packages](https://github.com/Stygian-Tech/latr-packages)

**Purpose:** **npm/Bun packages** for web and Node clients — lexicons, record keys, gateway client constants, and API contracts.

**Workspaces:**

| Package | Role |
|---------|------|
| `packages/lexicons` | JSON schemas for `com.latr.saved.external` / `com.latr.saved.item` |
| `packages/record-keys` | TypeScript deterministic rkeys + fingerprints; golden-vector tests |
| `packages/gateway-client` | Gateway header constants, OpenAPI drift tests |

**Also includes:**
- `openapi/latr-gateway.v1.yaml` — L@tr gateway OpenAPI stub
- `bruno/environments/local.bru` — local HTTP request examples

**Used by:** L@tr.link web app, The Social Wire web app (gateway client + key parity), any third-party L@tr client.

---

### [latr-reference](https://github.com/Stygian-Tech/latr-reference)

**Purpose:** **Self-hostable L@tr gateway** — the reference HTTP service that wraps `LatrKit` for save/list/archive/delete via `/v1/latr/*`.

**Contains:**
- Hummingbird `LatrGateway` executable
- OAuth + DPoP gate, client API key registry
- PDS write-through for `com.latr.saved.*` mutations
- Open Graph fetch + standard.site AT-URI discovery routes
- Docker + Fly deploy config (`fly.toml`, `deploy.sh`)
- `.env.example`, `scripts/dev.sh`, `scripts/doctor.sh`

**Used by:** Self-hosters, local dev, hosted L@tr.link (same codebase lineage).

**Depends on:** `latr-kit` (local path or future tagged release).

**Canonical API docs:** See `latr-link/docs/architecture/latr-gateway.md` in the L@tr monorepo.

---

## Social Wire domain packages

Everything related to **publication reading**, AppView projections, bootstrap stream, and Social Wire gateway routes — kept grouped while APIs are still stabilizing.

### [social-wire-appview-kit](https://github.com/Stygian-Tech/social-wire-appview-kit)

**Purpose:** **Experimental Swift packages** for AppView logic extracted from `ThinAppViewCore` and `services/appview`.

**Planned contents:**
- Publication projection and sidebar orchestration
- Bootstrap NDJSON stream service
- Read/enroll facades, pagination, cache store protocols
- Storage adapter interfaces (SQLite/Postgres — adapters stay in reference services)

**Status:** Scaffold only (`ThinAppViewCore` stub). APIs marked experimental until OpenAPI + fixtures stabilize.

**Depends on:** `atproto-primitives`.

---

### [social-wire-packages](https://github.com/Stygian-Tech/social-wire-packages)

**Purpose:** **npm/Bun SDK scaffolds** for Social Wire gateway and AppView HTTP contracts.

**Workspaces (experimental):**
- `gateway-client` — typed fetch helpers for `api.thesocialwire.app`
- `bootstrap-stream` — NDJSON `/v1/appview/bootstrap-stream` parser
- `api-models` — shared DTO types (future OpenAPI generation)

**Used by:** The Social Wire web app (eventually), third-party readers built on Social Wire APIs.

---

### [social-wire-reference](https://github.com/Stygian-Tech/social-wire-reference)

**Purpose:** **Self-hostable Social Wire backend** — gateway, AppView, and appview-worker as separate processes (matching production architecture).

**Contains (initial scaffold):**
- `docker-compose.yml` — three-service layout
- `.env.example` — gateway ↔ AppView trust, Supabase URL placeholders
- `scripts/dev.sh`, `scripts/doctor.sh`
- Service directories for gateway, appview, worker (to be filled from `the-social-wire` monorepo)

**Used by:** Self-hosters who want the full Social Wire read path without using hosted SaaS.

**Depends on:** `social-wire-appview-kit`, `gateway-trust-kit`, `atproto-auth-kit`, `federation-content-kit` (as extraction progresses).

---

## Meta / deployment

### [reference-deploy](https://github.com/Stygian-Tech/reference-deploy)

**Purpose:** **Deployment orchestration meta-repo** — pin released versions of reference services and validate self-hosting stacks.

**Contains:**
- `docker-compose.yml` — combined stack placeholder
- `scripts/smoke-test.sh` — health/OAuth/save smoke tests (configure URLs via env)
- This repository guide

**Used by:** Operators self-hosting multiple Stygian products, future CI integration tests across pinned releases.

**Does not contain:** Application source code — only deployment manifests and docs.

---

## How the monorepos relate

| Monorepo | Relationship to extracted repos |
|----------|-------------------------------|
| [latr-link](https://github.com/Stygian-Tech/latr-link) | Source of `latr-kit`, lexicons, and gateway; will eventually depend on published packages instead of in-tree copies |
| [the-social-wire](https://github.com/Stygian-Tech/the-social-wire) | Source of GatewayCore, ThinAppViewCore, AppView services; web app now calls L@tr via `latr-reference` for read-later mutations |

---

## Dependency rules (from the extraction plan)

1. **Foundation repos must not depend on product repos.**
2. **Kits must not import Hummingbird** unless they are gateway-oriented (`gateway-trust-kit` is the exception for HTTP helpers).
3. **Service repos compose kits + adapters** — DB, Fly secrets, and Supabase wiring stay in reference services, not in core packages.
4. **PDS records are canonical** for user data; AppView/Supabase/SQLite indexes are rebuildable derivatives.

---

## Local development quick reference

| Repo | Fast check |
|------|------------|
| Swift package | `scripts/bootstrap.sh && scripts/check.sh` |
| npm monorepo | `scripts/bootstrap.sh && bun test` |
| `latr-reference` | `scripts/dev.sh` (gateway on `:8080`) |
| Full stack (future) | `reference-deploy/docker-compose.yml` |

---

## GitHub links (all public)

- https://github.com/Stygian-Tech/atproto-primitives
- https://github.com/Stygian-Tech/atproto-auth-kit
- https://github.com/Stygian-Tech/gateway-trust-kit
- https://github.com/Stygian-Tech/federation-content-kit
- https://github.com/Stygian-Tech/mcp-server-kit
- https://github.com/Stygian-Tech/offline-sync-kit
- https://github.com/Stygian-Tech/latr-kit
- https://github.com/Stygian-Tech/latr-packages
- https://github.com/Stygian-Tech/latr-reference
- https://github.com/Stygian-Tech/social-wire-appview-kit
- https://github.com/Stygian-Tech/social-wire-packages
- https://github.com/Stygian-Tech/social-wire-reference
- https://github.com/Stygian-Tech/reference-deploy

---

## Provisioning script

To re-run GitHub repo creation/push for any new checkout:

```bash
bash "/Users/sam/Developer/Git/Stygian Tech/scripts/provision-github-repos.sh"
```

Set `STYGIAN_GITHUB_ORG` to override the default `Stygian-Tech` org.
