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

## Open Questions

- [ ] Script AST schema: finalize node types and metadata fields
- [ ] Series Bible Graph: define entity types, relationship types, and rule schema
- [ ] CRDT: Loro ProseMirror binding maturity — needs spike/POC
- [ ] AI: initial model selection and prompt architecture for Character Voice Registry
- [ ] Editorial: which NLEs to prioritize for round-trip (Avid vs Premiere vs Resolve)
- [ ] Deployment: single-tenant vs multi-tenant for enterprise clients
- [ ] Pricing: tier boundaries and AI usage metering approach
