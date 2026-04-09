# ScriptOS — Project Wiki

> **Design thesis:** Script-as-data, governed AI, orchestration-first workflows, and production-grade operational turnover.

ScriptOS is a unified platform for script writing, production planning, and studio operations. It treats the screenplay as structured, versioned data (Script AST) and propagates approved changes through breakdown, scheduling, budgeting, call sheets, on-set logging, editorial turnover, marketing, and compliance.

---

## Wiki Pages

| Page | Description |
|------|-------------|
| [01 — Product Vision & Differentiation](./01-product-vision.md) | Market positioning, capability comparison, north-star goals |
| [02 — Core Architecture](./02-core-architecture.md) | System layers, control plane, service mesh, deployment topology |
| [03 — Canonical Data Model](./03-data-model.md) | Script AST, Series Bible Graph, SeriesTimeline schemas |
| [04 — CRDT & Collaboration Strategy](./04-crdt-collaboration.md) | Loro vs Yjs analysis, tree move problem, semantic validation |
| [05 — AI Governance & Model Strategy](./05-ai-governance.md) | WGA compliance, model routing, AI ledger, cost modeling |
| [06 — Offline & On-Set Sync](./06-offline-sync.md) | PowerSync architecture, conflict resolution, Tauri client |
| [07 — Workflow Orchestration](./07-workflow-orchestration.md) | Temporal sagas, publish-to-production pipeline, compensation logic |
| [08 — Pre-Production & Production Modules](./08-production-modules.md) | Breakdown, scheduling, budgeting, call sheets, script supervisor |
| [09 — Post-Production & Editorial Round-Trip](./09-post-production.md) | OTIO integration, NLE sync, correlation database, dailies review |
| [10 — Search & Hybrid Retrieval](./10-search.md) | BM25 + vector search, RRF, Elasticsearch architecture |
| [11 — Security, Compliance & Accessibility](./11-security-compliance.md) | Watermarking, rights/NDA, audit trails, WCAG 2.2 AA |
| [12 — Infrastructure & Scaling](./12-infrastructure.md) | Graph DB choice, cost framework, scaling tiers, y-redis |
| [13 — Import & Migration Pipeline](./13-import-migration.md) | FDX, Fountain, PDF/OCR import, phased migration strategy |
| [14 — Architectural Decisions Log](./14-adr.md) | Key decisions made, rationale, and trade-offs |
| [15 — Implementation Roadmap](./15-roadmap.md) | Service priority, dependency chains, release plan |
| [16 — System Design Document](./16-system-design.md) | Service catalog, communication patterns, infra topology, auth, database ownership |

---

## Key Architectural Decisions (Summary)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| CRDT library | Loro (primary), Yjs (fallback) | Loro has native tree moves; Yjs has ecosystem maturity |
| Offline sync | PowerSync + SQLite | Battle-tested field service sync, Postgres-native |
| On-set client | Tauri 2.x | 3–10MB binary vs Electron's 85–165MB |
| Graph database | Neo4j | Cypher maps to bible traversals; GDS for story analytics |
| Search | Elasticsearch 8.9+ | Native RRF for hybrid BM25 + vector search |
| Orchestration | Temporal | Saga pattern for long-running approval workflows |
| AI models | Hybrid router (API) | Cheap models for simple tasks, quality for creative |
| Editorial interchange | OTIO + correlation DB | Metadata survives NLE round-trip via fallback matching |
| Rich text editor | ProseMirror / TipTap | Schema enforcement + CRDT binding maturity |
| CRDT sync scaling | y-redis | Memory-efficient streaming via Redis Streams |

---

## Decisions

**Script AST schema — finalized in wiki/03-data-model.md.**
All 19 node types, PostgreSQL DDL, revision color workflow, CRDT boundary, and export mappings. Closed.

**Series Bible Graph schema — finalized in wiki/03-data-model.md.**
13 Neo4j node labels, 25+ relationship types, Cypher constraints, vector index, fact provenance model. Closed.

**CRDT — Loro primary with in-house ProseMirror binding; Yjs fallback if POC fails.**
Decision recorded in wiki/04 and ADR-001/ADR-018. POC (Weeks 3–6) is the gate.

**AI model selection — Anthropic (Claude family) for generation; OpenAI text-embedding-3-large for embeddings.**
Model routing: Haiku for format/consistency checks, Sonnet for dialogue and scene analysis, Opus for coverage analysis. Prompt caching for Bible + voice context. See wiki/05 and ADR-020/ADR-024.

**NLE priority — Avid (AAF) first, DaVinci Resolve second, Premiere in v1.1.**
See wiki/09 and ADR-022.

**Multi-tenancy — multi-tenant SaaS by default; single-tenant VPC as enterprise add-on. See ADR-028.**
Multi-tenant SaaS (all customers on shared infrastructure, isolated by org_id + row-level security) is the default and covers the majority of clients. Single-tenant VPC (dedicated Postgres, Neo4j, and optional self-hosted AI) is available as a premium enterprise tier for studios requiring data sovereignty or custom AI deployment. Building single-tenant first would mean building the platform twice.

**Pricing tiers — Indie / Professional / Studio. See ADR-029.**

| Tier | Price | Includes |
|------|-------|---------|
| Indie | $49/seat/month | Editor, bible, collaboration. No AI generation. Limited breakdown (manual only). |
| Professional | $149/seat/month | Full AI assistance (500K tokens/month included), full breakdown + scheduling, FDX/PDF export. |
| Studio | Custom | Full platform, on-set module, watermarking, compliance reporting, self-hosted AI option, single-tenant VPC option. AI metered at cost + margin. |

AI metering for Studio: usage is tracked per org, billed monthly at inference cost + 30% margin. Overage for Professional tier: $0.005 per 1K tokens above the 500K monthly inclusion.
