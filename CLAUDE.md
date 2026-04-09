# CLAUDE.md — ScriptOS Project Context

> This file provides context for Claude Code (VS Code extension) to continue the architectural design and implementation of ScriptOS. It captures all decisions, strategies, and open questions from the initial design sessions.

## What is ScriptOS?

ScriptOS is a **unified platform for script writing, production planning, and studio operations**. It treats the screenplay as structured, versioned data (Script AST) and propagates approved changes through breakdown, scheduling, budgeting, call sheets, on-set logging, editorial turnover, marketing, and compliance.

**Design thesis:** Script-as-data, governed AI, orchestration-first workflows, and production-grade operational turnover.

## Project Wiki

The full architectural documentation lives in `wiki/`. Start with `wiki/00-index.md` for the table of contents.

Key pages:
- `wiki/03-data-model.md` — Script AST, Series Bible Graph, SeriesTimeline (START HERE for implementation)
- `wiki/04-crdt-collaboration.md` — Loro vs Yjs, tree move problem, semantic validation
- `wiki/05-ai-governance.md` — WGA compliance, model routing, AI ledger
- `wiki/06-offline-sync.md` — PowerSync, Tauri, on-set architecture
- `wiki/14-adr.md` — All architectural decisions with rationale

## Key Architectural Decisions

| Decision | Choice | Notes |
|----------|--------|-------|
| CRDT library | **Loro** (primary), Yjs (fallback) | Needs POC to validate ProseMirror binding maturity |
| Graph database | **Neo4j** (Aura managed) | For Series Bible — Cypher queries, GDS analytics |
| Search | **Elasticsearch 8.9+** | Native RRF for hybrid BM25 + vector search |
| Orchestration | **Temporal** | Saga pattern for publish-to-production and all multi-step workflows |
| Offline sync | **PowerSync** + SQLite | Postgres ↔ SQLite sync for on-set tools |
| On-set client | **Tauri 2.x** | 3–10MB binary; native SQLite; needs iOS maturity evaluation |
| CRDT scaling | **y-redis** | Memory-efficient streaming via Redis Streams; AGPL — needs commercial license |
| AI models | **Hybrid API router** | Cheap models for simple tasks, quality for creative; prompt caching for Bible context |
| Editorial interchange | **OTIO** + correlation DB | Metadata fallback matching by timecode + clip name + reel name |
| Editor framework | **ProseMirror / TipTap** | Schema enforcement + CRDT binding |
| Primary database | **PostgreSQL** | Scripts, users, projects, workflow state |
| Object storage | **S3-compatible** | Media, exports, backups, continuity photos |
| Cache + streams | **Redis** | CRDT streams, session cache, pub/sub |

## Implementation Sequence (Agreed)

The next engineering steps should follow this order:

### Step 1: Data Model (CURRENT PRIORITY)
Finalize the Script AST schema, Series Bible Graph schema, and SeriesTimeline schema. This is the spine — every module depends on it. See `wiki/03-data-model.md` for current state.

Open tasks:
- Define all AST node types with exact TypeScript interfaces and metadata fields
- Define Neo4j node labels and relationship types for Bible Graph
- Define revision color workflow mapping
- Define CRDT ↔ Postgres persistence boundary
- Define multi-format export from AST (FDX, Fountain, PDF)

### Step 2: System Design Document
Service boundaries, communication patterns (sync vs event-driven), orchestration layer, infrastructure topology. See `wiki/02-core-architecture.md` for architectural overview.

### Step 3: Technical Deep-Dives Per Module
In dependency order:
1. CRDT/collaboration layer (`wiki/04-crdt-collaboration.md`)
2. AI governance (`wiki/05-ai-governance.md`)
3. Offline sync (`wiki/06-offline-sync.md`)
4. Workflow orchestration (`wiki/07-workflow-orchestration.md`)
5. Editorial integration (`wiki/09-post-production.md`)

### Step 4: Implementation Roadmap
Sprint-level plans with dependency chains. See `wiki/15-roadmap.md` for current draft.

## Technology Stack (Decided)

### Backend
- **Language:** TypeScript (Node.js) or Rust (for performance-critical sync)
- **API:** gRPC for service-to-service, REST/GraphQL for client-facing
- **Database:** PostgreSQL (primary), Neo4j (graph), Elasticsearch (search), Redis (cache/streams)
- **Orchestration:** Temporal
- **Message bus:** Kafka or NATS (for event fan-out)
- **Object storage:** S3-compatible

### Frontend
- **Web:** React + TipTap (ProseMirror) + Tailwind
- **On-set:** Tauri 2.x with local SQLite
- **Mobile:** React Native (if Tauri mobile proves immature)

### Infrastructure
- **Container orchestration:** Kubernetes
- **Service mesh:** Istio or Linkerd
- **Observability:** OpenTelemetry + Prometheus + Grafana
- **CI/CD:** GitHub Actions or GitLab CI
- **Feature flags:** LaunchDarkly or Unleash

## Core Domain Concepts

### Script AST
The screenplay as a tree of typed nodes with stable UUIDs. Text content lives in CRDT text types at leaf level. Structural moves use tree CRDTs (Loro MovableTree) or coordinated operations (Yjs fallback).

### Series Bible Graph
Neo4j graph of characters, locations, lore, arcs, facts, rules, and voice profiles. Every fact has provenance (where it came from in the script). Used for AI grounding and continuity checking.

### SeriesTimeline
Maps story-day chronology across episodes. Continuity assertions (wardrobe, injuries, props) attach to story days, not individual scripts. Enables cross-episode continuity reasoning.

### Publish-to-Production Saga
Temporal workflow: lock draft → validate policy → extract breakdown → delta schedule → delta budget → route approvals → operational lock. Compensation on failure at each step.

### AI Contribution Ledger
Every AI interaction logged: timestamp, user, model, prompt hash, AI output, writer action (accepted/rejected/modified), modification delta, company consent reference. Required for WGA Article 72 compliance.

### Character Voice Registry
Per-character voice profiles constraining AI dialogue suggestions: vocabulary level, speech patterns, forbidden words, reference scenes for few-shot prompting.

## Constraints and Non-Negotiables

1. **WGA compliance** — AI is assistive only; never positions AI as author; every interaction logged
2. **Offline-first on-set** — Script Supervisor module must work fully offline for 12–16 hour days
3. **Industry-standard screenplay formatting** — Must match Final Draft output quality
4. **Forensic watermarking** — Every exported script carries invisible recipient-identifying watermark
5. **CRDT collaboration** — No central server dependency for text editing (structural ops may coordinate)
6. **Audit trail** — Every state mutation logged with actor, timestamp, and diff

## What Has NOT Been Decided

- [ ] Exact AST node types and field-level schemas (Step 1 priority)
- [ ] Neo4j node/relationship type definitions
- [ ] Backend language: TypeScript vs Rust vs hybrid
- [ ] Kubernetes provider: EKS vs GKE vs AKS
- [ ] Single-tenant vs multi-tenant for enterprise
- [ ] Pricing tiers and AI usage metering
- [ ] Loro vs Yjs (pending POC)
- [ ] Temporal Cloud vs self-hosted
- [ ] Which NLEs to prioritize for editorial round-trip
- [ ] Embedding model for vector search
- [ ] Tauri 2.x iOS readiness (needs evaluation)

## How to Use This File

When working on ScriptOS in Claude Code:
1. Read the relevant wiki page(s) before starting any module
2. Check `wiki/14-adr.md` for existing architectural decisions before proposing alternatives
3. Update wiki pages as decisions are made
4. Add new ADRs for significant technical choices
5. Keep the roadmap (`wiki/15-roadmap.md`) updated as tasks complete

## Project Structure (Proposed)

```
scriptos/
├── CLAUDE.md                    # This file
├── wiki/                        # Architecture documentation
│   ├── 00-index.md
│   ├── 01-product-vision.md
│   ├── ...
│   └── 15-roadmap.md
├── packages/                    # Monorepo packages
│   ├── ast/                     # Script AST types and utilities
│   ├── crdt/                    # CRDT layer (Loro/Yjs abstraction)
│   ├── editor/                  # ProseMirror/TipTap screenplay editor
│   ├── bible/                   # Series Bible service
│   ├── breakdown/               # Breakdown service
│   ├── scheduling/              # Scheduling service
│   ├── supervisor/              # Script Supervisor module
│   ├── ai/                      # AI governance + assistance
│   ├── import/                  # Import pipeline
│   ├── sync/                    # PowerSync offline layer
│   ├── orchestration/           # Temporal workflows
│   ├── search/                  # Elasticsearch integration
│   └── common/                  # Shared utilities, types, auth
├── apps/
│   ├── web/                     # React web application
│   ├── onset/                   # Tauri on-set application
│   └── api/                     # API gateway
├── infra/                       # Kubernetes, Terraform, Docker
└── scripts/                     # Dev tooling, migrations, seeds
```
