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

## Template for Future ADRs

```markdown
## ADR-NNN: [Title]

**Status:** Proposed | Accepted | Superseded by ADR-XXX

**Context:** [Why was this decision needed?]

**Decision:** [What was decided?]

**Rationale:** [Why this choice over alternatives?]

**Trade-offs:** [What was given up? What are the risks?]
```
