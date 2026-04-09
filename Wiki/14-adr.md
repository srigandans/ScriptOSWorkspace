# 14 — Architectural Decisions Log

## ADR Format

Each decision records: **Context** (why the decision was needed), **Decision** (what was chosen), **Rationale** (why), **Trade-offs** (what was given up), and **Status** (proposed/accepted/superseded).

---

## ADR-001: CRDT Library Selection

**Status:** Proposed

**Context:** ScriptOS needs real-time collaboration for both text editing and structural tree operations (scene moves, splits, merges). The tree move problem causes duplication with naive delete+insert CRDTs.

**Decision:** Loro as primary CRDT library, with Yjs + TipTap as fallback if Loro's ProseMirror binding proves immature.

**Rationale:** Loro is the only library with a native MovableTree CRDT implementing Kleppmann's formally verified algorithm. It also provides Peritext-compliant rich text. Yjs has far greater ecosystem maturity (900K+ npm weekly downloads) but lacks native tree moves.

**Trade-offs:** Loro's immaturity (smaller community, evolving API) vs Yjs's missing structural CRDT capabilities. The POC phase (4 weeks) will determine the final choice.

---

## ADR-002: Graph Database for Series Bible

**Status:** Accepted

**Context:** The Series Bible Graph stores characters, locations, lore, arcs, facts, and relationships with provenance. Needs traversal queries, analytics (PageRank for character importance), and managed service options.

**Decision:** Neo4j with Aura managed service.

**Rationale:** Cypher query language maps naturally to bible traversals. GDS library provides PageRank, community detection, and orphaned plot thread identification. Aura provides free tier → enterprise scaling. Largest developer ecosystem among graph databases.

**Trade-offs:** Rejected Dgraph (two acquisitions in two years — risky), TypeDB (interesting reasoning engine but immature ecosystem), and Amazon Neptune (viable for deep AWS shops but weaker developer experience).

---

## ADR-003: Offline Sync for On-Set Tools

**Status:** Accepted

**Context:** Film sets have poor/no connectivity. Script Supervisor module must work fully offline for 12–16 hour shooting days and sync reliably when connectivity returns.

**Decision:** PowerSync for structured data sync (Postgres ↔ SQLite), Loro/Yjs for rich text sync, Tauri 2.x for native client.

**Rationale:** PowerSync provides automatic offline write queuing, role-based Sync Streams, and has been battle-tested in Fortune 500 field service deployments. Tauri produces 3–10MB binaries vs Electron's 85–165MB.

**Trade-offs:** PowerSync is server-authoritative (acceptable for production workflows). ElectricSQL was considered but lacks bidirectional CRDT sync in v1.0. cr-sqlite has the most elegant architecture but declining project activity.

---

## ADR-004: Workflow Orchestration Engine

**Status:** Accepted

**Context:** Production workflows are long-running (hours to weeks), require compensation logic, and must survive service restarts. Examples: publish-to-production saga, import pipeline, call sheet generation.

**Decision:** Temporal for all multi-step approval and compensation workflows.

**Rationale:** Durable execution guarantees. Workflows survive infrastructure failures. Built-in retry, timeout, and compensation. Excellent visibility into in-flight workflows.

**Trade-offs:** Operational complexity of running Temporal. Alternative: Temporal Cloud (managed) to reduce ops burden. Non-orchestrated events use standard event bus (Kafka/NATS).

---

## ADR-005: Search Infrastructure

**Status:** Accepted

**Context:** Platform needs hybrid search combining keyword (BM25) and semantic (vector kNN) retrieval for scripts, bible entries, and production data.

**Decision:** Elasticsearch 8.9+ with native Reciprocal Rank Fusion (RRF).

**Rationale:** Single system for both BM25 and vector search. Built-in RRF eliminates manual score normalization. 1-second refresh latency. Sufficient for ScriptOS's scale (even 10K+ screenplays). Separate vector databases only make sense at 100M+ vectors.

**Trade-offs:** Elasticsearch operational complexity. Alternative: OpenSearch (AWS-managed, API-compatible) if on AWS.

---

## ADR-006: AI Model Strategy

**Status:** Proposed

**Context:** AI assistance must be WGA-compliant, cost-efficient, and support both SaaS and enterprise (data-residency) deployments.

**Decision:** Phase 1: API-based multi-model routing (Claude/GPT family). Phase 2: Self-hosted option (Llama/Mistral on vLLM). Phase 3: BYOK adapter for studios with existing AI contracts.

**Rationale:** API-based launch is fastest. Model routing (cheap for simple tasks, quality for creative) reduces costs 50–70%. Prompt caching (especially Series Bible context) reduces costs further ~90% on cached hits. Self-hosted addresses studio data-residency requirements.

**Trade-offs:** API-based means data leaves the perimeter — some studios won't accept this. Self-hosted requires MLOps investment ($10K–50K/month GPU infrastructure).

---

## ADR-007: Editorial Interchange Format

**Status:** Accepted

**Context:** Need bidirectional sync with NLEs (Avid, Premiere, Resolve). Custom metadata is routinely stripped by NLEs during round-trip.

**Decision:** OTIO as primary interchange format, with AAF for Avid and EDL for universal compatibility. Correlation database as mandatory fallback for metadata recovery.

**Rationale:** OTIO provides human-readable JSON, supports arbitrary metadata namespaces, and has adapters for AAF/FCPXML/EDL. The correlation database (matching by timecode + clip name + reel name) recovers script references even when NLE-specific metadata is lost.

**Trade-offs:** OTIO doesn't preserve complex NLE constructs (effects, compound clips, multicam). Manual NLE export required (no push API from NLEs).

---

## ADR-008: Client Framework for On-Set App

**Status:** Proposed

**Context:** On-set tools need native performance, offline SQLite access, small binary size, and cross-platform support (iPad, desktop, potentially Android).

**Decision:** Tauri 2.x for desktop and mobile.

**Rationale:** 3–10MB binary size (vs Electron 85–165MB). Native SQLite access. Cross-platform including iOS/Android in Tauri 2.x. Rust backend for performance-critical sync operations.

**Trade-offs:** Tauri 2.x mobile support is newer than React Native. May need React Native fallback for iOS if Tauri mobile proves immature. Needs evaluation during POC phase.

---

## ADR-009: Event Bus — NATS JetStream

**Status:** Accepted

**Context:** Services need async decoupled communication for fan-out (search indexing, notifications, audit logging, continuity checks). Decision was between Kafka and NATS JetStream.

**Decision:** NATS JetStream for all async event fan-out.

**Rationale:** ScriptOS's event throughput is modest (thousands/day, not millions/second). NATS JetStream provides persistence, consumer groups, and replay at dramatically lower operational cost than Kafka. Kafka requires brokers, ZooKeeper/KRaft, partition management — significant overhead for this scale. NATS runs as a single binary and clusters trivially.

**Trade-offs:** If ScriptOS reaches massive multi-tenant scale (10,000+ concurrent writers), NATS may require migration to Kafka. Topic contracts are kept consistent to make this migration mechanical, not architectural.

---

## ADR-010: Client API — GraphQL Federation

**Status:** Accepted

**Context:** Clients need to compose data from many services in single requests (script + breakdown + continuity + story days). REST would require multiple round-trips or a BFF layer that duplicates service logic.

**Decision:** GraphQL Federation (Apollo Router + per-service subgraphs) for all client-facing data queries and mutations. REST retained for: file uploads, AI streaming (SSE), webhook delivery, OIDC/SAML flows, SCIM.

**Rationale:** The AST is inherently hierarchical; GraphQL models this naturally. Federation lets each service own its type definitions without a central schema monolith. Apollo Router handles query planning and cross-service entity resolution via `@key` directives.

**Trade-offs:** Federation adds operational complexity (subgraph registration, schema composition checks in CI). Apollo Router is a critical path component — requires high-availability deployment.

---

## ADR-011: Internal Communication — gRPC

**Status:** Accepted

**Context:** Service-to-service calls need strongly typed contracts, efficient serialization, and streaming support (for AI embedding generation, CRDT checkpoint flows).

**Decision:** gRPC with Protocol Buffers for all internal service-to-service calls.

**Rationale:** Proto contracts fail at compilation rather than runtime — prevents interface drift. Binary protocol is efficient. HTTP/2 multiplexing reduces connection overhead. Code generation for TypeScript is mature. Temporal activities are gRPC calls wrapped in durable execution.

**Trade-offs:** Proto schema management adds discipline requirement (proto files as shared contracts, versioned). Debugging gRPC traffic is less immediate than REST (requires tooling like grpcurl or Postman gRPC support).

---

## ADR-012: Auth Delegation — Centralized JWT Validation at Gateway

**Status:** Accepted

**Context:** All services need authenticated user context. Options: each service validates JWT independently, or the API gateway validates once and injects claims.

**Decision:** HTTP Gateway validates JWT via Auth Service gRPC on every request. Injects `X-User-ID`, `X-Org-ID`, `X-Roles` headers. Internal services trust these headers; no JWT parsing in application services.

**Rationale:** Eliminates JWT library maintenance in every service. Istio mTLS ensures only the gateway can inject these headers — a service impersonating the gateway is impossible inside the mesh. Simplifies security surface area.

**Trade-offs:** API Gateway is now a critical path for auth. Gateway degradation means all services fail auth. Mitigation: multi-pod gateway with circuit breaker that serves cached permission results for 60 seconds during Auth Service degradation.

---

## ADR-013: Database Isolation — Schema-per-Service on Shared Cluster

**Status:** Accepted

**Context:** Services need data isolation (no service reads another's tables directly) but deploying a separate Postgres cluster per service adds significant operational burden at launch.

**Decision:** Each service domain gets a dedicated PostgreSQL schema within a single managed Postgres cluster. Each service uses a dedicated DB user with GRANT access only to its own schema. Cross-schema Postgres FKs are prohibited — cross-service references are application-level UUIDs.

**Rationale:** Enforces service boundaries at the DB user level without the operational cost of multiple clusters. Schema-level isolation is the logical boundary needed for future extraction. A service's schema can be migrated to its own cluster when it needs independent scaling or fault isolation.

**Trade-offs:** Shared cluster means one service's bad query can degrade others (mitigated by per-service connection pools with max connections, and read replicas for heavy readers). Not suitable if services need different Postgres versions or extension sets.

---

## ADR-014: Temporal Deployment — Temporal Cloud (Managed)

**Status:** Accepted

**Context:** Temporal Server (persistence, history, matching, frontend services) is operationally complex to run self-hosted. Decision: self-hosted vs Temporal Cloud.

**Decision:** Temporal Cloud for the server. Temporal Workers (workflow and activity code) run as internal Kubernetes deployments within the service mesh.

**Rationale:** Temporal Server failure modes (storage replication, worker node management) are eliminated. Temporal Cloud provides 99.99% SLA. Cost at ScriptOS launch volume (hundreds of workflows/day) is modest. Workers running inside the mesh means all activity calls benefit from mTLS — no additional auth needed for Temporal → service communication.

**Trade-offs:** Temporal Cloud means workflow state leaves the perimeter. Not acceptable for air-gapped enterprise deployments. For enterprise customers requiring on-premises deployment, a self-hosted Temporal option must be available (same worker code, different server connection config).

---

## ADR-015: Two-Gateway Architecture

**Status:** Accepted

**Context:** CRDT collaboration uses long-lived WebSocket connections (~50KB memory each, hours-long duration). HTTP requests are short-lived and CPU-intensive. Mixing on one gateway creates resource contention and prevents independent scaling.

**Decision:** Two separate gateway deployments: (1) HTTP Gateway (Apollo Router) for GraphQL, REST, and auth flows; (2) WebSocket (Collaboration) Gateway for CRDT sync and in-app notification push.

**Rationale:** WebSocket connections scale based on active editing sessions — independent of HTTP request volume. HPA triggers differ: HTTP scales on CPU/latency, WebSocket scales on connection count. Separate deployments allow separate resource limits and independent rollout.

**Trade-offs:** Clients must maintain two connection points. Connection upgrade (HTTP → WebSocket) must be routed to the correct gateway. Handled by a single DNS entry with path-based routing at the Kubernetes Ingress layer.

---

## ADR-016: On-Set Sync Boundary — PowerSync Manages Supervisor Data

**Status:** Accepted

**Context:** On-set devices need structured data sync with offline-first capability. Supervisor Service data (take logs, setups, deviations) must be available offline and sync back when connectivity returns.

**Decision:** PowerSync manages all structured data sync between cloud Postgres (Supervisor schema) and on-set SQLite. The Tauri app never uses gRPC or GraphQL for Supervisor data — it reads/writes SQLite, and PowerSync handles bidirectional sync. Script content (ast_nodes) is synced to on-set as read-only. Continuity photos upload directly to S3 via presigned URLs when connectivity is available.

**Rationale:** PowerSync handles offline write queuing, conflict resolution rules, and role-based sync streams without custom sync infrastructure. Separating photo upload (S3 direct) from structured data sync (PowerSync) keeps the sync layer focused and avoids overloading it with binary blobs.

**Trade-offs:** Two sync mechanisms (PowerSync + S3 presigned URL) instead of one. PowerSync self-hosted licensing required for basecamp deployments (remote locations without internet). Cost TBD.

---

## ADR-017: Repository Strategy — Monorepo

**Status:** Accepted

**Context:** ScriptOS has 15 microservices, 4 Temporal worker deployments, 2 client apps (web + Tauri), and a shared layer of Proto files, NATS event payloads, AST types, and Temporal workflow definitions. The team needed to decide between a monorepo and a polyrepo (one repo per service).

**Decision:** Single monorepo managed with pnpm workspaces + Turborepo (TypeScript) and a Cargo workspace (Rust/Tauri). Services are deployed as separate container images — monorepo is a development-time concept only.

**Rationale:** Four forcing functions make polyrepo untenable:
1. **Proto files** — gRPC contracts shared between caller and implementer must be a single source of truth. A `proto` package in the monorepo is one import; in polyrepo it's a versioned npm release on every API change.
2. **NATS event payload types** — `DomainEvent<T>` payloads shared across up to 5 consumers. Type drift across repos is inevitable and causes silent runtime failures.
3. **AST types** — `ASTNode`, `RevisionColor`, `BreakdownTag` used by Script Service, Breakdown, Import, the web editor, and the Tauri app. Publishing these as internal npm packages adds friction with no benefit.
4. **Temporal activity contracts** — workflow definitions and activity implementations must have identical type signatures; a shared `packages/workflows` makes this a compile-time guarantee.

Turborepo's affected task graph ensures CI only rebuilds and redeploys the packages touched by a given PR — a change to `services/script/` does not rebuild all 15 services.

**Monorepo structure:**
```
scriptos/
├── packages/       ← shared: ast, proto, events, workflows, db, common
├── services/       ← 15 microservices (each has own Dockerfile)
├── workers/        ← Temporal workers (publish-saga, import-saga, call-sheet, revision-dist)
├── apps/           ← web (React), onset (Tauri), gateway (Apollo Router config)
├── infra/          ← Kubernetes, Terraform, Helm
├── Cargo.toml      ← Cargo workspace root (Rust crates in apps/onset/src-tauri)
├── package.json    ← pnpm workspace root
└── turbo.json      ← Turborepo pipeline config
```

**Trade-offs:** Monorepos require discipline on dependency direction (services must not import from other services — only from `packages/`). Large repos can have slower git operations at scale (mitigated by git sparse-checkout if needed). CI pipeline complexity increases slightly vs. per-repo pipelines, but Turborepo's caching offsets this with faster builds.

---

## ADR-018: Loro ProseMirror Binding — Build In-House, Contribute Upstream

**Status:** Accepted

**Context:** Loro has no mature ProseMirror binding. Y-ProseMirror (the Yjs equivalent) is battle-tested but tightly coupled to Yjs. ScriptOS needs a binding to connect the ProseMirror editor to Loro's text and tree CRDTs.

**Decision:** Build a thin in-house binding layer. Map ProseMirror transactions → Loro ops and Loro ops → ProseMirror steps. Once battle-tested internally, open-source the binding to accelerate Loro ecosystem adoption.

**Rationale:** We cannot wait for upstream — Loro's community is small and the binding timeline is unknown. The binding is a thin translation layer (not a complex algorithm), so building it in-house is tractable. Open-sourcing it after internal validation reduces our maintenance burden by gaining community contributors, and follows the same path Y-ProseMirror took.

**Trade-offs:** We carry the binding's maintenance until it gains community adoption. Risk: if Loro's API changes significantly, our binding breaks. Mitigation: pin Loro version, update deliberately.

---

## ADR-019: CRDT Structural Ops — Decompose Into Atomic Tree Ops

**Status:** Accepted

**Context:** Scene splits and merges are common screenplay operations. A split creates a new scene from the second half of an existing one. A merge combines two adjacent scenes. Options were: (a) implement a custom CRDT operation in Loro, or (b) decompose into existing atomic Loro MovableTree operations.

**Decision:** Decompose split/merge into a serialized sequence of atomic Loro MovableTree ops: create node → move children → update metadata → soft-delete shell. The Redis coordination lock (`struct_lock:{script_id}`) serializes these sequences to prevent interleaving.

**Rationale:** A custom CRDT op requires patching Loro's formally-verified Rust core — a maintenance liability we cannot afford with a small team against an evolving library. Decomposition into atomic ops stays entirely within Loro's existing semantics. The coordination lock ensures the sequence is atomic from the perspective of other clients.

**Trade-offs:** The coordination lock means split/merge requires connectivity (cannot be done offline without potential conflict). Acceptable: structural edits are intentional, deliberate operations — unlike text edits which need offline support. Writers do not split scenes while offline on set.

---

## ADR-020: AI Context Assembly — Hierarchical Prompt Caching

**Status:** Accepted

**Context:** AI assistance requires assembling Bible context, voice profiles, timeline state, and surrounding scene text into each request. This context is large (potentially 10K–50K tokens) and expensive if sent fresh each time.

**Decision:** Hierarchical context with Anthropic prompt caching. Stable prefix (Bible facts for relevant entities, voice profile, world rules, timeline state) is the cacheable section. Variable suffix (current scene + surrounding 2–3 scenes) is appended fresh per request. Cache the stable prefix via Anthropic's cache_control API.

**Rationale:** Anthropic cache hits cost $0.30/M tokens vs $3.00/M for misses — 90% cost reduction on the context portion. The stable prefix accounts for ~90% of the total tokens in a typical request. The variable section must be fresh (current scene changes as the writer edits). This design collapses the per-request cost from ~$0.15 to ~$0.02 for a typical creative assistance request.

**Trade-offs:** Cache invalidation: if a writer adds a new Bible fact mid-session, the cached prefix is stale. Mitigation: invalidate and rebuild the cache when Bible facts change (tracked via `bible.fact_created` event). The first request after invalidation pays full price; subsequent requests hit the cache.

---

## ADR-021: Scheduling Solver — Google OR-Tools

**Status:** Accepted

**Context:** The stripboard scheduling problem is a constraint satisfaction problem: assign scenes to shoot days while respecting actor availability, location grouping, union rules, and day/night balance. Options: build a custom solver, use OR-Tools (Google, Apache 2.0), or integrate a commercial scheduling product.

**Decision:** Google OR-Tools as the optimization engine wrapped by the Scheduling Service. Engineers write constraints in the OR-Tools API; the solver finds the optimal or near-optimal schedule. Manual 1st AD override is always available.

**Rationale:** Building a constraint solver from scratch is a research project that could consume months. OR-Tools is a proven, production-grade solver (Apache 2.0, used in logistics and scheduling at major companies). It handles all required constraint types natively. Commercial scheduling products (Movie Magic Scheduling, StudioBinder) don't offer embeddable APIs and would require round-trip file exchange, not integration.

**Trade-offs:** OR-Tools adds a Python or Java runtime dependency (depending on which SDK is used). If using the Python API, the Scheduling Service needs a Python sidecar or subprocess. Alternatively, the Go or Java OR-Tools SDK avoids Python. Decision: use the OR-Tools Java SDK with a JVM sidecar — more operationally familiar for the team than Python for a latency-sensitive service.

---

## ADR-022: NLE Priority — Avid First, Resolve Second, Premiere in v1.1

**Status:** Accepted

**Context:** ScriptOS must support editorial round-trip with NLEs. Supporting all three simultaneously (Avid, Resolve, Premiere) at launch is too broad. A priority order was needed.

**Decision:** Avid Media Composer (AAF format) first. DaVinci Resolve (native OTIO) second. Premiere Pro (FCPXML) in v1.1.

**Rationale:** High-budget TV drama and feature films — the primary market — run on Avid. Avid requires AAF specifically; OTIO alone is insufficient for Avid workflows. DaVinci Resolve has native OTIO support (the lowest engineering effort). Premiere is common in digital/streaming but is lower priority for the target client profile at launch.

**Trade-offs:** Premiere users cannot use the editorial round-trip at launch. Mitigation: Premiere users can still use the correlation database for manual re-import; they lack the automated diff flow.

---

## ADR-023: Dailies Review — Frame.io Connector First

**Status:** Accepted

**Context:** ScriptOS needs a dailies review surface. Options: build a native lightweight player, or integrate with Frame.io (industry standard).

**Decision:** Frame.io connector at launch (v1). Native lightweight player with script-linked annotations in v1.1.

**Rationale:** Frame.io is the industry standard for dailies review. Productions already have Frame.io workflows and accounts. The connector (OAuth integration, comment webhook, take reference linking) delivers real value immediately without competing with an established tool. A native player becomes worthwhile when we can offer features Frame.io doesn't: AI-assisted take selection based on circled takes + continuity photos, script-linked comment threading, and automatic linked script marker packages.

**Trade-offs:** Dependency on Frame.io's API availability and pricing. Mitigation: the native player in v1.1 is the fallback path and differentiates from Frame.io when ready.

---

## ADR-024: Embedding Model — OpenAI text-embedding-3-large

**Status:** Accepted

**Context:** ScriptOS needs embeddings for semantic scene search, Bible fact similarity, and continuity conflict detection. Options: Voyage-3, OpenAI text-embedding-3-large, or self-hosted (BGE-M3, e5-large).

**Decision:** OpenAI `text-embedding-3-large` (1536 dimensions) at launch. Self-hosted BGE-M3 or e5-large-v2 via vLLM for enterprise data-residency deployments (OpenAI-compatible endpoint, no code changes).

**Rationale:** text-embedding-3-large leads MTEB retrieval benchmarks for the relevant tasks and is already in use by the majority of teams in the OpenAI ecosystem. At ScriptOS's scale, the cost difference between Voyage-3 and OpenAI is negligible. Simplicity: one embedding vendor managed alongside the generation vendor (Anthropic). Enterprise self-hosted path is clean because vLLM exposes an OpenAI-compatible API.

**Trade-offs:** OpenAI dependency for embeddings. If OpenAI pricing increases or the model is deprecated, migration to Voyage-3 or self-hosted is straightforward (all vectors need re-embedding, which is a one-time bulk operation). The embedding abstraction layer in the AI Service makes this mechanical.

---

## ADR-025: Watermarking — In-House Steganographic; Digimarc for Enterprise Litigation

**Status:** Accepted

**Context:** Every exported script must carry an invisible forensic watermark encoding the recipient's identity. Options: build in-house steganographic techniques, or license Digimarc (certified forensic watermarking used in major studios).

**Decision:** In-house watermarking for all tiers using: (1) character spacing micro-adjustments in PDF (imperceptible, survives print and scan), (2) Unicode zero-width character injection in FDX text (survives copy-paste). Digimarc available as a premium enterprise add-on for clients who need legally certified provenance for litigation.

**Rationale:** Digimarc licensing is expensive and creates a hard external dependency for all clients regardless of need. In-house techniques are well-understood, adequate for leak identification, and have no per-export cost. Digimarc's legal certification is only needed when the watermark evidence is presented in court — a requirement for major studio clients willing to pay the premium.

**Trade-offs:** In-house watermarks may be more susceptible to sophisticated removal attacks (transcription, reformatting) than Digimarc's perceptual hashing. Acceptable for the majority of leak scenarios (accidental sharing, insider leaks). For studios where the threat model includes sophisticated adversaries with motivation to defeat the watermark, Digimarc is the answer.

---

## ADR-026: Kubernetes Provider — GKE Autopilot

**Status:** Accepted

**Context:** ScriptOS needs a Kubernetes provider for production deployment. Options: GKE (Google), EKS (AWS), AKS (Azure).

**Decision:** GKE with Autopilot mode as the default production deployment. EKS offered as an enterprise deployment option for clients with AWS procurement requirements.

**Rationale:** GKE Autopilot manages node provisioning, security patching, and cluster upgrades automatically — critical for a small team. Google Cloud's native integrations (Cloud SQL, Artifact Registry, Cloud Armor) reduce operational surface area. Kubernetes was created at Google — GKE has deepest feature set and earliest access to new capabilities. GKE Autopilot's per-pod billing model aligns cost with actual resource usage.

**Trade-offs:** Vendor lock-in to GCP ecosystem. Mitigation: service code is cloud-agnostic; only infrastructure configuration (Terraform) is GCP-specific. EKS equivalent Terraform is maintained as a parallel module for enterprise deployments.

---

## ADR-027: Database Migrations — Atlas (ariga.io)

**Status:** Accepted

**Context:** 13 PostgreSQL schemas (one per service) require migration tooling. Options: Flyway, Liquibase, Atlas.

**Decision:** Atlas (ariga.io) with declarative schema definitions.

**Rationale:** Atlas generates migration SQL from schema declarations (`schema.hcl` files) — engineers declare the target state, Atlas computes the diff. This approach scales cleanly across 13 schemas without requiring engineers to hand-write SQL migrations for every change. Atlas handles drift detection (flags when a running database diverges from its declared schema), which is critical in a multi-schema environment. Flyway and Liquibase require hand-written migrations — acceptable for 1–2 schemas but error-prone at this count.

**Trade-offs:** Atlas is newer and has a smaller community than Flyway. The declarative approach can surprise engineers accustomed to imperative migrations — some complex migrations (data backfills, multi-step schema changes) still require custom SQL. Atlas supports escape hatches for these cases.

---

## ADR-028: Multi-Tenancy — Multi-Tenant SaaS Default; Single-Tenant VPC Enterprise Add-On

**Status:** Accepted

**Context:** Enterprise studio clients may require data sovereignty, custom AI deployment, or dedicated infrastructure. Options: build single-tenant from the start, or build multi-tenant with a single-tenant upgrade path.

**Decision:** Multi-tenant SaaS as the default (all clients on shared infrastructure, isolated by org_id + row-level security). Single-tenant VPC (dedicated Postgres, Neo4j, AI inference) available as a premium enterprise tier.

**Rationale:** Building single-tenant first means building the platform twice — the multi-tenant SaaS and the enterprise deployment infrastructure simultaneously, before any revenue. Multi-tenant SaaS ships faster, serves the majority of clients, and generates revenue. The schema-per-service database design (ADR-013) and org_id scoping in all tables makes the multi → single-tenant migration path mechanical: extract each service's schema to a dedicated cluster, update connection strings. Single-tenant is an operational deployment variant, not a different codebase.

**Trade-offs:** Studios with hard data residency requirements (some government-adjacent productions) cannot use the multi-tenant SaaS at all. They must wait for the enterprise single-tenant offering. Prioritize single-tenant development once the multi-tenant platform has revenue to fund it.

---

## ADR-029: Pricing — Three Tiers (Indie / Professional / Studio)

**Status:** Accepted

**Context:** ScriptOS spans use cases from solo indie writers to major studio productions. A single pricing tier either undercharges enterprises or overcharges independents.

**Decision:** Three tiers:

| Tier | Price | Scope |
|------|-------|-------|
| **Indie** | $49/seat/month | Editor, Bible, collaboration, FDX/PDF export. No AI generation. Manual breakdown only. |
| **Professional** | $149/seat/month | Full AI assistance (500K tokens/month included), full breakdown + scheduling, all export formats. |
| **Studio** | Custom contract | Full platform, on-set module, watermarking, compliance reporting (WGA AI ledger export), single-tenant VPC option, self-hosted AI option. AI metered at cost + 30% margin. |

**Rationale:** Aligns with the target personas: Indie writers don't need production modules. Professional writers/showrunners need AI and planning. Studios need compliance, watermarking, and on-set integration. The AI token inclusion in Professional (500K tokens/month ≈ 15 full screenplay analyses) is generous enough to cover typical usage but creates a clear upgrade path to Studio for heavy AI users.

**Trade-offs:** Custom pricing for Studio creates sales complexity. Mitigation: publish a Studio pricing floor (e.g., minimum $2,000/month) to qualify inbound leads.

---

## Template for Future ADRs

```markdown
## ADR-NNN: [Title]

**Status:** Proposed | Accepted | Superseded by ADR-XXX

**Context:** [Why was this decision needed?]

**Decision:** [What was decided?]

**Rationale:** [Why this choice over alternatives?]

**Trade-offs:** [What was given up? What are the risks?]
```
