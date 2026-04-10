# CLAUDE.md — ScriptOS Project Context

> This file provides context for Claude Code (VS Code extension) to continue the architectural design and implementation of ScriptOS. It is kept in sync with the wiki — when a decision is made, both CLAUDE.md and the relevant wiki page are updated.

## What is ScriptOS?

ScriptOS is a **unified platform for script writing, production planning, and studio operations**. It treats the screenplay as structured, versioned data (Script AST) and propagates approved changes through breakdown, scheduling, budgeting, call sheets, on-set logging, editorial turnover, marketing, and compliance.

**Design thesis:** Script-as-data, governed AI, orchestration-first workflows, and production-grade operational turnover.

## Project Wiki

The full architectural documentation lives in `Wiki/`. Start with `Wiki/00-index.md` for the table of contents.

Key pages (read in this order when onboarding):
- `Wiki/03-data-model.md` — **FINALIZED** Script AST (19 node types), Series Bible Graph (Neo4j), SeriesTimeline, PostgreSQL DDL, export mappings
- `Wiki/16-system-design.md` — **FINALIZED** Service catalog (15 services), communication patterns, API gateway, event bus, auth, database ownership, infra topology, monorepo structure
- `Wiki/14-adr.md` — All 29 architectural decisions with rationale (ADR-001 through ADR-029)
- `Wiki/04-crdt-collaboration.md` — Loro vs Yjs, tree move problem, binding strategy, compaction
- `Wiki/05-ai-governance.md` — WGA compliance, model routing, prompt caching, AI ledger
- `Wiki/06-offline-sync.md` — PowerSync, Tauri, on-set architecture, SQLCipher key management
- `Wiki/07-workflow-orchestration.md` — Temporal sagas, approval routing, escalation policy
- `Wiki/08-production-modules.md` — Breakdown (OR-Tools), scheduling, budgeting, call sheets
- `Wiki/09-post-production.md` — OTIO/AAF/NLE round-trip (Avid first), Frame.io connector
- `Wiki/11-security-compliance.md` — Watermarking, SOC 2 timeline, GDPR, accessibility
- `Wiki/12-infrastructure.md` — GKE Autopilot, Atlas migrations, y-redis licensing, S3 vs R2
- `Wiki/15-roadmap.md` — Phased rollout plan, team size assumptions, milestone commitments

## Engineering Progress

| Step | Description | Status |
|------|-------------|--------|
| **Step 1** | Canonical Data Model (AST, Bible Graph, SeriesTimeline) | ✅ COMPLETE |
| **Step 2** | System Design Document (service boundaries, comms, infra) | ✅ COMPLETE |
| **Step 3** | Technical Deep-Dives per module (CRDT → AI → Offline → Orchestration → Editorial) | ✅ COMPLETE |
| **Step 4** | Implementation Roadmap (sprint-level plans) | ✅ COMPLETE |

**Step 3 sequence:** ~~CRDT/collaboration layer~~ ✅ → ~~AI governance~~ ✅ → ~~Offline sync~~ ✅ → ~~Workflow orchestration~~ ✅ → ~~Editorial integration~~ ✅. All complete.

## Key Architectural Decisions (All Closed)

| Decision | Choice | ADR |
|----------|--------|-----|
| CRDT library | **Loro** (primary) + in-house ProseMirror binding; Yjs fallback | ADR-001, ADR-018 |
| CRDT structural ops | Decompose into atomic MovableTree ops + Redis coordination lock | ADR-019 |
| CRDT scaling | **Hocuspocus** (startup); **y-redis** (growth, AGPL commercial license) | ADR-012 (infra) |
| Graph database | **Neo4j Aura** managed | ADR-002 |
| Search | **Elasticsearch 8.9+** — native RRF for BM25 + vector search | ADR-005 |
| Embedding model | **OpenAI text-embedding-3-large**; self-hosted BGE-M3 for enterprise | ADR-024 |
| Orchestration | **Temporal Cloud** (managed server); workers inside K8s mesh | ADR-004, ADR-014 |
| Offline sync | **PowerSync** + SQLite (WAL + SQLCipher) | ADR-003 |
| On-set client | **Tauri 2.x** desktop at launch; iOS evaluated Q3 2026 | ADR-008 |
| Editorial interchange | **OTIO** + AAF (Avid first) + correlation DB | ADR-007, ADR-022 |
| Dailies review | **Frame.io connector** (v1); native player (v1.1) | ADR-023 |
| Editor framework | **ProseMirror / TipTap** | ADR-001 |
| Primary database | **PostgreSQL** — schema-per-service on shared cluster; Atlas migrations | ADR-013, ADR-027 |
| Graph database | **Neo4j Aura** | ADR-002 |
| Object storage | **AWS S3** (evaluate R2 if egress > $500/month) | ADR-026 |
| Cache + streams | **Redis** — CRDT streams, session cache, pub/sub | — |
| Event bus | **NATS JetStream** (not Kafka) | ADR-009 |
| Client API | **GraphQL Federation** (Apollo Router); REST for upload/SSE/webhooks | ADR-010 |
| Internal comms | **gRPC** (Protocol Buffers) | ADR-011 |
| Auth boundary | JWT validated at HTTP Gateway only; headers injected into service mesh | ADR-012 |
| Gateways | Two: **HTTP Gateway** (Apollo Router) + **WebSocket** (Collaboration Gateway) | ADR-015 |
| Kubernetes | **GKE Autopilot** (default); EKS for enterprise AWS clients | ADR-026 |
| Service mesh | **Istio** (mTLS, traffic management, observability) | — |
| CI/CD | **GitHub Actions** + Turborepo affected graph | — |
| Feature flags | **Unleash** (self-hosted, open source) | — |
| Repository | **Monorepo** — pnpm workspaces + Turborepo + Cargo workspace | ADR-017 |
| Scheduling solver | **Google OR-Tools** (Apache 2.0) wrapped by Scheduling Service | ADR-021 |
| AI models | **Anthropic** (Claude family) for generation; prompt caching for Bible context | ADR-020 |
| AI self-hosted | **Llama 3.1 70B** via vLLM for enterprise data-residency | ADR-006 |
| Watermarking | **In-house steganographic** (char spacing + zero-width chars); Digimarc add-on for enterprise | ADR-025 |
| Multi-tenancy | **Multi-tenant SaaS** default; single-tenant VPC as enterprise add-on | ADR-028 |
| Pricing | **Indie** $49/seat · **Professional** $149/seat · **Studio** custom | ADR-029 |
| NLE priority | **Avid** (AAF) → **DaVinci Resolve** (OTIO) → Premiere (v1.1) | ADR-022 |

## Technology Stack (Finalized)

### Backend
- **Language:** TypeScript (Node.js) for all services; Rust for Tauri on-set backend
- **API (client-facing):** GraphQL Federation (Apollo Router) + REST for file upload, SSE, webhooks
- **API (internal):** gRPC with Protocol Buffers
- **Databases:** PostgreSQL (primary, schema-per-service), Neo4j Aura (Bible Graph), Elasticsearch 8.9+ (search + vectors), Redis (CRDT streams + cache + pub/sub)
- **Orchestration:** Temporal Cloud (server) + Temporal Workers (in K8s)
- **Event bus:** NATS JetStream
- **Object storage:** AWS S3
- **Migrations:** Atlas (ariga.io) declarative schema management
- **Scheduling solver:** Google OR-Tools (Java SDK)

### Frontend
- **Web:** React + TipTap (ProseMirror) + Tailwind CSS
- **On-set:** Tauri 2.x (Rust backend + React frontend) — desktop first; iOS Q3 2026
- **Mobile fallback:** React Native (if Tauri iOS proves immature)

### Infrastructure
- **Container orchestration:** Kubernetes — GKE Autopilot (default)
- **Service mesh:** Istio (mTLS, traffic management)
- **Observability:** OpenTelemetry + Prometheus + Grafana + Jaeger/Tempo + Loki
- **CI/CD:** GitHub Actions + Turborepo remote cache
- **Feature flags:** Unleash (self-hosted)
- **Monorepo:** pnpm workspaces + Turborepo (TS) + Cargo workspace (Rust)

## Monorepo Structure

```
scriptos/
├── packages/                    # Shared libraries (imported by services + apps)
│   ├── ast/                     # ASTNode types, RevisionColor, BreakdownTag
│   ├── proto/                   # .proto files + generated TypeScript gRPC stubs
│   ├── events/                  # NATS DomainEvent<T> envelope + payload types
│   ├── workflows/               # Temporal workflow definitions + activity interfaces
│   ├── db/                      # Atlas schema files, shared query helpers
│   └── common/                  # Auth header utils, OpenTelemetry setup, logger
├── services/                    # 15 microservices (each has own Dockerfile)
│   ├── script/                  # Script AST, versions, revision workflow
│   ├── collaboration/           # CRDT sync (WebSocket + y-redis)
│   ├── bible/                   # Neo4j Series Bible Graph
│   ├── breakdown/               # NLP element detection, breakdown catalog
│   ├── scheduling/              # Stripboard, shoot days, OR-Tools solver
│   ├── budget/                  # Budget lines, account codes, exports
│   ├── continuity/              # SeriesTimeline, assertions, violations
│   ├── supervisor/              # On-set take logs, deviations, turnover
│   ├── ai/                      # Model routing, AI ledger, consent, semantic cache
│   ├── import/                  # FDX/Fountain/PDF/OCR import pipeline
│   ├── search/                  # Elasticsearch indexing + hybrid search
│   ├── watermark/               # Forensic watermark embed/extract
│   ├── legal/                   # Rights tags, NDA gates, legal holds
│   ├── notification/            # Email + WebSocket push + mobile push
│   └── auth/                    # OIDC/SAML, SCIM, JWT, RBAC
├── workers/                     # Temporal workers (separate K8s deployments)
│   ├── publish-saga/
│   ├── import-saga/
│   ├── call-sheet/
│   └── revision-dist/
├── apps/
│   ├── web/                     # React web app
│   ├── onset/                   # Tauri on-set app (src-tauri/ + src/)
│   └── gateway/                 # Apollo Router config + supergraph schema
├── infra/                       # Kubernetes manifests, Terraform, Helm charts
├── Cargo.toml                   # Cargo workspace root
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
└── turbo.json                   # Turborepo pipeline config
```

**Dependency rule:** `services/*` may only import from `packages/*`. Services must never import from other services — cross-service data goes through gRPC APIs only. Enforced via ESLint `import/no-restricted-paths` in CI.

## Core Domain Concepts

### Script AST
19 node types (project → script/episode → act → scene → heading/action/dialogue/etc.) with stable UUID v7 identities. Text lives in CRDT text types at leaf nodes. Structural moves use Loro MovableTree. Full schema: `Wiki/03-data-model.md`.

### Series Bible Graph
Neo4j graph of characters, locations, props, vehicles, organizations, story events, facts, rules, arcs, lore, and voice profiles. Every fact has provenance (source scene, confidence, supersedure chain). Used for AI grounding, continuity checking, and character voice enforcement.

### SeriesTimeline
Story-day chronology across episodes. Continuity assertions (wardrobe, injuries, props, VFX) attach to story days + time-of-day slots, not individual scripts. Cross-episode continuity violations detected by AI + surfaced to Script Supervisor.

### Publish-to-Production Saga
Temporal workflow: lock draft → validate policy (Legal) → validate canon (Bible) → extract breakdown → delta schedule → delta budget → route approvals (parallel for independent depts, sequential for dependent) → operational lock. Auto-escalate at 24h; auto-approve at 48h. Full spec: `Wiki/07-workflow-orchestration.md`.

### AI Contribution Ledger
Every AI interaction logged immutably: timestamp, user, model, prompt hash, context refs, AI output, writer action (accepted/rejected/modified), modification delta, company consent reference. Required for WGA Article 72 compliance. Rate limited per org (quota) and per user (soft cap).

### Character Voice Registry
Per-character voice profiles (vocabulary level, speech patterns, forbidden words, 3–5 reference scenes for few-shot prompting). AI dialogue suggestions are checked against the registry before display. Violations are warnings — never blocks.

## Non-Negotiables

1. **WGA compliance** — AI is assistive only; every interaction logged; opt-in; company consent tracked
2. **Offline-first on-set** — Script Supervisor module fully offline for 12–16 hour days; 7-day design target
3. **Industry-standard screenplay formatting** — Final Draft output quality; FDX round-trip fidelity
4. **Forensic watermarking** — Every exported script carries invisible recipient-identifying watermark
5. **CRDT collaboration** — Text editing offline-capable; structural ops require connectivity
6. **Audit trail** — Every state mutation logged (actor, timestamp, diff) in immutable Audit Service
7. **All decisions documented** — Every architectural/design decision written into wiki with rationale + ADR if significant

## How to Use This File

When working on ScriptOS in Claude Code:
1. **Read `Wiki/00-index.md`** for a full table of contents before starting any module
2. **Check `Wiki/14-adr.md`** (ADR-001 through ADR-029) before proposing any alternative to a closed decision
3. **Read the relevant wiki page** before writing any code or design for a module
4. **Update both the wiki page AND this file** whenever a decision is made — open questions must be closed with rationale, not left hanging
5. **Add a new ADR** for any decision that is hard to reverse, affects multiple services, or represents a non-obvious choice
6. **All 4 steps are complete** — implementation begins with Sprint 0 per `Wiki/15-roadmap.md`
