# 16 — System Design Document

> **Status:** FINALIZED — v1.0 (2026-04-09)
> This document closes the open questions from the wiki and establishes the authoritative service boundaries, communication contracts, and infrastructure topology for ScriptOS. All downstream deep-dives (wiki pages 04–15) should be read against this document.

## Table of Contents

1. [Decisions Made](#1-decisions-made)
2. [Service Catalog](#2-service-catalog)
3. [Communication Architecture](#3-communication-architecture)
4. [API Gateway Design](#4-api-gateway-design)
5. [Event Bus — NATS JetStream](#5-event-bus--nats-jetstream)
6. [Temporal Orchestration Placement](#6-temporal-orchestration-placement)
7. [Auth & RBAC](#7-auth--rbac)
8. [Database Ownership Model](#8-database-ownership-model)
9. [CRDT Collaboration Architecture](#9-crdt-collaboration-architecture)
10. [On-Set Architecture (PowerSync Boundary)](#10-on-set-architecture-powersync-boundary)
11. [Infrastructure Topology](#11-infrastructure-topology)
12. [Observability](#12-observability)
13. [Repository Structure](#13-repository-structure)
14. [Service Dependency Map](#14-service-dependency-map)
15. [Open Items for Deep-Dives](#15-open-items-for-deep-dives)

---

## 1. Decisions Made

The following questions were open across the wiki pages. They are closed here with rationale. Engineering must not re-litigate these decisions without an ADR update.

### 1.1 Event Bus: NATS JetStream (not Kafka)

**Closed.** NATS JetStream provides message persistence, consumer groups, replay from sequence offset, and at-least-once delivery. ScriptOS's event throughput (dozens of concurrent writers, thousands of events per day) is orders of magnitude below Kafka's design point. Kafka's operational overhead (brokers, ZooKeeper/KRaft, partition rebalancing) is not justified at this scale. NATS JetStream runs as a single binary, integrates with Kubernetes natively, and can migrate to a clustered deployment if throughput demands it.

**If wrong:** Scale to NATS cluster first (trivial). If NATS itself becomes a bottleneck (unlikely before 10,000 concurrent writers), migrate to Kafka behind the same topic contracts.

### 1.2 Client-Facing API: GraphQL Federation

**Closed.** The client needs to compose data across services in a single request — script nodes + breakdown elements + continuity assertions + story days. REST forces multiple round-trips or a BFF (backend for frontend) that duplicates service logic. GraphQL Federation (Apollo Router + subgraphs) lets each service own its type definitions and resolvers while the router composes them at query time.

REST is retained for: file uploads (multipart), AI streaming responses (Server-Sent Events), webhook delivery, and PowerSync sync endpoints.

### 1.3 Internal Service Communication: gRPC

**Closed.** All service-to-service calls use gRPC with Protocol Buffers. Reasons: strongly typed contracts that fail at compilation rather than runtime, binary protocol efficiency, bidirectional streaming for the collaboration checkpoint flow, and first-class code generation for TypeScript. The API Gateway translates between GraphQL (external) and gRPC (internal).

### 1.4 Auth: Centralized at the API Gateway

**Closed.** The API Gateway validates every JWT before it enters the service mesh. On success, it injects three headers:
- `X-User-ID`: authenticated user UUID
- `X-Org-ID`: organization UUID
- `X-Roles`: comma-separated RBAC roles

Internal services trust these headers without re-validating the token. The service mesh (Istio mTLS) ensures that only the gateway can inject these headers — traffic from outside the mesh without valid certificates is dropped. No JWT library lives in application services.

**Exception:** The Auth Service itself (issues and validates tokens; does not trust injected headers from other services).

### 1.5 Breakdown ↔ Script: No Direct Service Call

**Closed.** The Breakdown Service never calls the Script Service directly, and the Script Service never calls the Breakdown Service directly.

- **Incremental updates:** Script Service publishes `ScriptCheckpointed`. Breakdown Service subscribes and re-runs NLP detection on changed scenes.
- **Publish saga:** Temporal orchestrates the publish workflow; Temporal activities call each service independently. The ordering (Script lock → Breakdown → Schedule → Budget) is managed by Temporal's workflow logic, not by services calling each other.

### 1.6 Database Boundaries: Schema-per-Service on One Cluster

**Closed.** Each service domain owns a dedicated PostgreSQL schema within a single managed PostgreSQL cluster. Cross-schema Postgres foreign keys are prohibited — cross-service references are application-level UUIDs only. When a service needs to query another service's data, it either calls the service's gRPC API or reads from an event-derived local copy.

This choice prioritizes operational simplicity for the initial build. Extraction to separate database clusters (for independent scaling, fault isolation, or compliance requirements) is straightforward — schemas are already the isolation boundary.

### 1.7 Two Gateways: HTTP and WebSocket

**Closed.** CRDT collaboration has a fundamentally different scaling profile from HTTP request handling. WebSocket connections are long-lived (hours) and memory-intensive (~50KB each). HTTP requests are short-lived and CPU-intensive. Mixing them on the same gateway creates resource contention and complicates independent scaling.

- **HTTP Gateway:** Handles GraphQL, REST (upload, SSE), and webhook routing. Auth validation, rate limiting, and request logging live here.
- **WebSocket (Collaboration) Gateway:** Handles CRDT sync only. Auth validation for initial handshake only. Scales based on active editing sessions.

### 1.8 Temporal: Cloud-Managed, Workers Inside the Mesh

**Closed.** Temporal Server runs on Temporal Cloud (managed). This eliminates the operational burden of running Temporal Server (storage, replication, worker node management) which is significant. Temporal Workers — the processes that execute workflow and activity code — run as internal Kubernetes deployments within the service mesh, calling other services via gRPC with mTLS.

**Cost consideration:** Temporal Cloud pricing is per workflow action. ScriptOS's workflow volume (publish sagas, import jobs, call sheet generation) is modest — well within cost-effective range at launch.

### 1.9 Notification Delivery: Three-Channel

**Closed.** The Notification Service delivers via:
1. **In-app WebSocket:** Reuses the Collaboration Gateway WebSocket infrastructure. Notification events are pushed to connected clients on their personal channel (`user:{user_id}`).
2. **Email:** Transactional email via SendGrid (fallback: AWS SES).
3. **Mobile push:** FCM (Android) + APNs (iOS) via a unified push abstraction.

### 1.10 On-Set PowerSync Boundary

**Closed.** PowerSync manages all structured data synchronization between the cloud Postgres and on-set SQLite. The on-set Tauri app does not use gRPC or GraphQL for operational data — it reads and writes SQLite, and PowerSync handles the sync. Direct API calls from on-set devices are limited to: file upload for continuity photos (via S3 presigned URL), and connection health checks.

---

## 2. Service Catalog

Fifteen services in total. Each entry defines: **responsibility**, **data ownership**, **external API**, **internal gRPC API**, **events published**, **events consumed**, and **called by Temporal as**.

---

### Script Service

**Responsibility:** Source of truth for all screenplay data. Owns the project tree, script metadata, and the AST node adjacency list. Manages version snapshots, revision workflow state, and scene locks.

**Data ownership:**
- PostgreSQL schema `script`: `projects`, `scripts`, `ast_nodes`, `script_versions`

**GraphQL subgraph (via API Gateway):**
- `Project`, `Script`, `Season`, `Episode`, `ASTNode`, `ScriptVersion` types
- Queries: `project`, `script`, `scene`, `scriptVersion`, `projects` (paginated)
- Mutations: `createProject`, `createScript`, `updateSceneMetadata`, `lockDraft`, `unlockDraft`, `createVersion`

**gRPC API (internal):**
```
ScriptService {
  GetScript(ScriptRequest) → ScriptResponse
  GetSceneSnapshot(SceneRequest) → SceneSnapshotResponse
  CheckpointCRDT(CheckpointRequest) → CheckpointResponse
  LockDraft(LockRequest) → LockResponse
  UnlockDraft(UnlockRequest) → UnlockResponse
  ApplyOperationalLock(LockRequest) → LockResponse
  GetVersionSnapshot(VersionRequest) → VersionResponse
  GetProjectScripts(ProjectRequest) → ScriptsResponse
}
```

**Events published** (to NATS JetStream):
- `script.created` — project and script IDs, format, title
- `script.checkpointed` — script ID, list of dirty node IDs, checkpoint timestamp
- `script.published` — script ID, version ID, revision color
- `script.locked` — script ID, locked by, locked at
- `scene.modified` — scene ID, script ID, which metadata fields changed
- `script.imported` — script ID, import provenance reference

**Events consumed:** None. Script Service is the upstream source of truth; it does not react to other services' events.

**Called by Temporal as:**
- `lock_draft`, `unlock_draft`, `apply_operational_lock`, `get_snapshot_for_breakdown`

---

### Collaboration Service

**Responsibility:** Real-time CRDT synchronization for concurrent screenplay editing. Manages WebSocket connections, routes CRDT update operations through Redis Streams, and triggers checkpoints to the Script Service when sessions go idle.

**Data ownership:**
- Redis Streams: `crdt:{script_id}` — encoded CRDT update operations (Loro/Yjs ops)
- No Postgres tables (stateless between sessions)

**WebSocket API (via Collaboration Gateway, not HTTP Gateway):**
```
ws://collab.scriptos.internal/sync/{script_id}

Handshake: Bearer JWT in Authorization header (one-time validation)
Messages: binary-encoded CRDT update operations
Subprotocol: scriptos-crdt-v1
```

**gRPC API (internal):** None. The Collaboration Service is only reachable via WebSocket from clients.

**Internal calls made:**
- Script Service gRPC `CheckpointCRDT` — on idle timeout or explicit save
- Presence: tracks connected users per document in Redis hash (ephemeral)

**Events published:** None.
**Events consumed:** None.

**Scaling:** Horizontally stateless via y-redis. Any number of WebSocket server instances can handle connections for any document. Redis Streams are the coordination layer.

---

### Series Bible Service

**Responsibility:** Canonical knowledge graph for the series — characters, locations, props, vehicles, organizations, story events, facts, rules, arcs, lore, and voice profiles. Provides AI grounding context and conflict detection.

**Data ownership:**
- Neo4j database: all Bible Graph entities and relationships (see `wiki/03-data-model.md §3`)

**GraphQL subgraph:**
- `Character`, `Location`, `Prop`, `Vehicle`, `Organization`, `StoryEvent`, `Fact`, `Rule`, `Arc`, `LoreEntry`, `VoiceProfile` types
- Queries: `character`, `characters`, `location`, `fact`, `searchBible`, `checkConflicts`, `getVoiceProfile`
- Mutations: `createCharacter`, `addFact`, `supersedeFact`, `createArc`, `updateVoiceProfile`, `linkCharacterToScene`

**gRPC API (internal):**
```
BibleService {
  GetCharacter(CharacterRequest) → CharacterResponse
  GetContextForAI(AIContextRequest) → AIContextResponse
    // Assembles: character facts + voice profile + arc state + location facts
    // for the AI Service to use as grounding context
  CheckConflicts(ConflictCheckRequest) → ConflictCheckResponse
  GetVoiceProfile(VoiceProfileRequest) → VoiceProfileResponse
  ExtractAndUpsertFacts(SceneContentRequest) → ExtractedFactsResponse
    // NLP-based fact extraction from scene content_snapshot
  GetCharactersByScene(SceneRequest) → CharactersResponse
}
```

**Events published:**
- `bible.fact_created` — entity ID, fact ID, category, statement, confidence
- `bible.conflict_detected` — entity ID, conflicting fact IDs, conflict description
- `bible.character_voice_updated` — character ID, changed fields

**Events consumed:**
- `scene.modified` → calls `ExtractAndUpsertFacts` on changed scenes asynchronously

**Called by Temporal as:**
- `validate_canon` (checks new script version for unresolved bible conflicts before publish)

---

### Breakdown Service

**Responsibility:** Auto-detects and manages production breakdown elements (cast, props, stunts, VFX, etc.) from the Script AST. Provides the element catalog used by Scheduling and Budgeting.

**Data ownership:**
- PostgreSQL schema `breakdown`: `breakdown_elements`

**GraphQL subgraph:**
- `BreakdownElement`, `BreakdownScene` types
- Queries: `breakdownForScene`, `breakdownForScript`, `breakdownForVersion`, `elementSearch`
- Mutations: `confirmElement`, `addElement`, `removeElement`, `updateElement`

**gRPC API (internal):**
```
BreakdownService {
  GetBreakdownForScene(SceneRequest) → BreakdownResponse
  GetBreakdownForScript(ScriptRequest) → BreakdownResponse
  ExtractBreakdown(ExtractionRequest) → ExtractionResponse
    // Full NLP extraction for a locked script version
    // Called by Temporal during publish saga
  GetElementsByCategory(CategoryRequest) → ElementsResponse
}
```

**Events published:**
- `breakdown.extracted` — script ID, version ID, element count by category
- `breakdown.element_updated` — scene ID, element ID, change type

**Events consumed:**
- `script.checkpointed` → re-runs incremental NLP detection on dirty scene IDs

**Called by Temporal as:**
- `extract_breakdown` (full extraction in publish saga), `rollback_breakdown` (compensation)

**NLP pipeline note:** The NLP detection logic within Breakdown Service calls a dedicated internal NLP inference endpoint (served via Python/FastAPI sidecar or shared ML inference service). This is not a separate microservice — it is a subprocess or co-deployed container within the Breakdown Service deployment.

---

### Scheduling Service

**Responsibility:** Manages the shooting schedule (stripboard) — shoot days, scene assignments, location grouping, and day-out-of-days. Receives breakdown data and produces the schedule that drives call sheet generation and budget.

**Data ownership:**
- PostgreSQL schema `scheduling`: `shoot_days`, `scheduled_scenes`, `schedule_versions`

**GraphQL subgraph:**
- `ShootDay`, `ScheduledScene`, `ScheduleVersion`, `DayOutOfDays` types
- Queries: `schedule`, `shootDay`, `dayOutOfDays`, `sceneShootDay`
- Mutations: `createShootDay`, `assignSceneToShootDay`, `reorderShootDay`, `lockSchedule`

**gRPC API (internal):**
```
SchedulingService {
  ComputeScheduleDelta(DeltaRequest) → DeltaResponse
    // Given a new breakdown version, compute which shoot days are affected
  GetScheduleForScript(ScriptRequest) → ScheduleResponse
  GetShootDayScenes(ShootDayRequest) → ScenesResponse
    // Used by Call Sheet Service
  LockScheduleVersion(LockRequest) → LockResponse
}
```

**Events published:**
- `schedule.updated` — script ID, version ID, affected shoot day IDs
- `schedule.shoot_day_changed` — shoot day ID, added/removed scene IDs

**Events consumed:**
- `breakdown.extracted` → flags scenes that need re-scheduling (no auto-reschedule — human reviews and adjusts)

**Called by Temporal as:**
- `compute_schedule_delta`, `rollback_schedule`

---

### Budgeting Service

**Responsibility:** Manages budget lines, account codes, fringes, and department rollups. Budget versions are linked to specific schedule + breakdown versions. Produces exports for EP Budgeting, Movie Magic, Excel, and PDF.

**Data ownership:**
- PostgreSQL schema `budget`: `budget_versions`, `budget_lines`, `account_codes`, `department_rollups`

**GraphQL subgraph:**
- `BudgetVersion`, `BudgetLine`, `AccountCode`, `DepartmentRollup` types
- Queries: `budget`, `budgetLine`, `departmentSummary`
- Mutations: `createBudgetVersion`, `updateBudgetLine`, `applyFringes`

**gRPC API (internal):**
```
BudgetService {
  ComputeBudgetDelta(DeltaRequest) → DeltaResponse
  ExportBudget(ExportRequest) → ExportResponse  // triggers async export job
  GetBudgetSummary(BudgetRequest) → SummaryResponse
}
```

**Events published:**
- `budget.updated` — budget version ID, changed department IDs

**Events consumed:**
- `schedule.updated` → flags budget lines for recalculation (does not auto-recalculate — human reviews)

**Called by Temporal as:**
- `compute_budget_delta`, `rollback_budget`

---

### AI Service

**Responsibility:** All AI-assisted features: dialogue suggestions, consistency checks, structure analysis, coverage analysis, character voice validation. Manages the AI Contribution Ledger and model routing. Enforces WGA compliance controls.

**Data ownership:**
- PostgreSQL schema `ai`: `ai_ledger_entries`, `consent_records`, `semantic_cache`

**REST API (via HTTP Gateway, SSE for streaming):**
- `POST /ai/suggest` — returns SSE stream of token chunks
- `POST /ai/check` — returns JSON result (non-streaming)
- `GET /ai/ledger` — paginated ledger for WGA reporting
- `POST /ai/consent` — record company consent for a project

**gRPC API (internal):**
```
AIService {
  Suggest(SuggestRequest) → stream SuggestChunk
  CheckVoice(VoiceCheckRequest) → VoiceCheckResponse
  CheckConsistency(ConsistencyRequest) → ConsistencyResponse
  AnalyzeStructure(AnalysisRequest) → AnalysisResponse
  GenerateEmbedding(EmbeddingRequest) → EmbeddingResponse
    // Used by Search Service for indexing
}
```

**Events published:** None (AI Service is request-response only).

**Events consumed:** None.

**Internal calls made:**
- Bible Service gRPC `GetContextForAI` — to assemble grounding context
- Bible Service gRPC `GetVoiceProfile` — for character voice validation
- Script Service gRPC `GetSceneSnapshot` — for surrounding scene context

**Rate limiting:** Enforced internally per user ID + org ID. Separate limits per interaction type (creative generation vs. analysis). Limits configurable per pricing tier.

**Model routing table (at launch):**

| Task | Tier 1 (Haiku/GPT-4o mini) | Tier 2 (Sonnet/GPT-4o) | Tier 3 (Opus) |
|------|---------------------------|------------------------|----------------|
| Format correction | ✅ | — | — |
| Consistency check | ✅ | — | — |
| Voice check | ✅ | — | — |
| Dialogue suggestion | — | ✅ | — |
| Scene analysis | — | ✅ | — |
| Coverage analysis | — | — | ✅ |

---

### Import Pipeline Service

**Responsibility:** Parses FDX, Fountain, PDF (digital and OCR), and Celtx files into the canonical Script AST. Manages import jobs, confidence scoring, and source provenance records. Preserves the original file in S3.

**Data ownership:**
- PostgreSQL schema `import`: `import_jobs`, `import_provenance`, `import_warnings`
- S3 prefix `imports/`: original uploaded files

**REST API (via HTTP Gateway, multipart upload):**
- `POST /import/upload` — multipart file upload, returns `job_id`
- `GET /import/job/{job_id}` — status, warnings, and preview
- `POST /import/job/{job_id}/accept` — triggers Temporal Import Saga to persist AST

**gRPC API (internal, called by Temporal activities):**
```
ImportService {
  DetectFormat(FileRequest) → FormatResponse
  ParseToAST(ParseRequest) → ASTResponse
  ValidateAST(ASTRequest) → ValidationResponse
  PersistAST(PersistRequest) → PersistResponse
    // Calls Script Service gRPC to create projects/scripts/ast_nodes
}
```

**Events published:**
- `import.completed` — job ID, script ID, overall confidence, warning count
- `import.failed` — job ID, error reason

**Events consumed:** None.

**Called by Temporal as:**
- `detect_format`, `parse_to_ast`, `validate_ast`, `generate_confidence_report`, `persist_ast`

---

### Script Supervisor Service

**Responsibility:** On-set data capture — take logs, setups, line deviations, continuity photos, and daily turnover reports. The authoritative source for what was actually shot. Data is PowerSync-synced to on-set devices.

**Data ownership:**
- PostgreSQL schema `supervisor`: `setups`, `take_logs`, `line_deviations`, `turnover_reports`
- S3 prefix `continuity-photos/`: photo assets

**GraphQL subgraph:**
- `Setup`, `TakeLog`, `LineDeviation`, `TurnoverReport` types
- Queries: `takesForScene`, `setupsForShootDay`, `dailyReport`
- Mutations: `createSetup`, `logTake`, `recordDeviation`, `generateTurnoverReport`

**PowerSync sync streams** (to on-set SQLite, role-scoped):
- Script Supervisor: reads `ast_nodes` (scene content), `scheduled_scenes`, `continuity_assertions`; writes `setups`, `take_logs`, `line_deviations`
- 1st AD: reads `scheduled_scenes`, `shoot_days`; writes scene completion status
- Editorial: reads `take_logs` (circled/printed status only)

**gRPC API (internal):**
```
SupervisorService {
  GetTakesForScene(SceneRequest) → TakesResponse
  GetDailyWrap(ShootDayRequest) → WrapResponse
    // Used by Turnover Saga to generate lined script + facing pages
  GetDeviationsForScene(SceneRequest) → DeviationsResponse
}
```

**Events published:**
- `supervisor.take_logged` — scene ID, take ID, circled boolean, timecodes
- `supervisor.wrap_reported` — shoot day ID, scenes covered, pages shot

**Events consumed:** None (on-set captures reality, doesn't react to upstream systems during a shoot).

---

### Continuity Service

**Responsibility:** Manages the SeriesTimeline — story days, time-of-day slots, continuity assertions, and violation tracking. Runs cross-episode continuity checks when scenes are modified.

**Data ownership:**
- PostgreSQL schema `continuity`: `story_days`, `time_of_day_slots`, `scene_timeline_map`, `continuity_assertions`, `assertion_scene_map`, `continuity_violations`

**GraphQL subgraph:**
- `StoryDay`, `TimeOfDaySlot`, `ContinuityAssertion`, `ContinuityViolation` types
- Queries: `timeline`, `assertionsForScene`, `openViolations`, `storyDay`
- Mutations: `createStoryDay`, `mapSceneToStoryDay`, `createAssertion`, `resolveViolation`

**gRPC API (internal):**
```
ContinuityService {
  CheckContinuity(CheckRequest) → CheckResponse
    // Runs AI-assisted continuity check on a scene against prior story day assertions
  GetTimeline(ProjectRequest) → TimelineResponse
  GetAssertionsForScene(SceneRequest) → AssertionsResponse
  GetStoryDay(StoryDayRequest) → StoryDayResponse
}
```

**Events published:**
- `continuity.violation_detected` — assertion ID, scene ID, violation description, severity
- `continuity.assertion_updated` — assertion ID, new status

**Events consumed:**
- `scene.modified` → runs `CheckContinuity` for modified scenes (async, debounced)
- `script.checkpointed` → runs `CheckContinuity` for all scenes with story_day assignments that have changed

**Internal calls made:**
- AI Service gRPC `CheckConsistency` — for the AI-assisted continuity check step

---

### Search Service

**Responsibility:** Indexes script content, bible entities, and production data into Elasticsearch. Serves hybrid BM25 + vector kNN search with RRF ranking to all clients.

**Data ownership:**
- Elasticsearch indexes: `scriptos-scenes`, `scriptos-bible`, `scriptos-production`

**REST API (via HTTP Gateway):**
- `GET /search?q=...&type=scenes|bible|all&project_id=...` — hybrid search
- `GET /search/suggest?q=...` — autocomplete for character names and locations

**gRPC API (internal):**
```
SearchService {
  IndexScene(IndexRequest) → IndexResponse
  IndexBibleEntity(IndexRequest) → IndexResponse
  DeleteFromIndex(DeleteRequest) → DeleteResponse
}
```

**Events published:** None.

**Events consumed:**
- `script.checkpointed` → indexes updated scene content_snapshots + metadata
- `script.published` → re-indexes all scenes in the version (ensures locked version is indexed)
- `bible.fact_created` → indexes the new fact
- `bible.conflict_detected` → updates conflict status in scene index

**Internal calls made:**
- AI Service gRPC `GenerateEmbedding` — for vector embedding generation during indexing

---

### Watermarking Service

**Responsibility:** Embeds and extracts invisible forensic watermarks from exported scripts (PDF, FDX). Records every export with recipient identity and access scope.

**Data ownership:**
- PostgreSQL schema `watermark`: `watermark_records`, `export_log`

**gRPC API (internal, no client-facing API):**
```
WatermarkService {
  EmbedWatermark(EmbedRequest) → WatermarkedDocument
    // Input: script_version_id, recipient_id, access_scope
    // Output: bytes of watermarked PDF or FDX
  ExtractWatermark(ExtractRequest) → WatermarkMetadata
    // For leak investigation — takes uploaded file, returns encoded identity
  GetExportLog(ExportLogRequest) → ExportLogResponse
}
```

**Events published:**
- `watermark.export_recorded` — recipient ID, script version ID, export timestamp, access scope

**Events consumed:** None.

**Called by:**
- Script Service (inline during export pipeline — not via Temporal)

---

### Legal / Rights Service

**Responsibility:** Rights tags, NDA gates, release tracking, legal hold flags, and compliance reporting. Enforces distribution restrictions at the delivery boundary.

**Data ownership:**
- PostgreSQL schema `legal`: `rights_tags`, `nda_records`, `release_records`, `legal_holds`, `compliance_reports`

**GraphQL subgraph:**
- `RightsTag`, `NDARecord`, `ReleaseRecord`, `LegalHold` types
- Queries: `rightsForScript`, `ndaStatus`, `legalHoldStatus`
- Mutations: `applyRightsTag`, `recordNDAAcceptance`, `createLegalHold`, `liftLegalHold`

**gRPC API (internal):**
```
LegalService {
  CheckDistributionRights(RightsRequest) → RightsResponse
    // Is this recipient authorized to receive this script version?
  ValidatePolicy(PolicyRequest) → PolicyResponse
    // Pre-publish: checks AI disclosure, NDA conditions, active legal holds
  GetNDAStatus(NDARequest) → NDAResponse
}
```

**Events published:**
- `legal.hold_created` — project ID, reason
- `legal.nda_accepted` — recipient ID, project ID

**Events consumed:** None.

**Called by Temporal as:**
- `validate_policy` (in publish saga step 2)

---

### Notification Service

**Responsibility:** Delivers notifications to users via in-app WebSocket, email, and mobile push. Pure event consumer — no domain logic, no persistent state beyond delivery records.

**Data ownership:**
- PostgreSQL schema `notify`: `notification_log` (delivery audit only — what was sent, when, to whom)

**No gRPC API.** Purely event-driven.

**Delivery channels:**
- **In-app:** Sends message to user's WebSocket channel (`user:{user_id}`) via the Collaboration Gateway's Redis pub/sub
- **Email:** Transactional email via SendGrid API
- **Mobile push:** Firebase Cloud Messaging (Android) + APNs (iOS) via unified push adapter

**Events consumed:**
- `script.published` → notify all project collaborators
- `continuity.violation_detected` → notify Script Supervisor + assigned writer
- `import.completed` → notify user who triggered import
- `import.failed` → notify user who triggered import
- `workflow.approval_required` → notify approver(s)
- `workflow.approved` → notify requester
- `workflow.denied` → notify requester
- `supervisor.wrap_reported` → notify editorial and post supervisor

---

### Auth Service

**Responsibility:** Identity, authentication, and authorization. Issues JWTs, handles OIDC/SAML flows for enterprise SSO, manages SCIM provisioning for org user sync.

**Data ownership:**
- PostgreSQL schema `auth`: `users`, `organizations`, `memberships`, `roles`, `permissions`, `sessions`, `audit_tokens`

**REST API (external-facing, via HTTP Gateway):**
- OAuth 2.0 / OIDC flows: `/auth/authorize`, `/auth/token`, `/auth/refresh`, `/auth/logout`
- SAML SP endpoints: `/auth/saml/sso`, `/auth/saml/callback`
- SCIM 2.0: `/scim/v2/Users`, `/scim/v2/Groups`

**gRPC API (internal):**
```
AuthService {
  ValidateToken(TokenRequest) → ClaimsResponse
    // Called by API Gateway on every request
  GetUser(UserRequest) → UserResponse
  CheckPermission(PermissionRequest) → PermissionResponse
    // For fine-grained permission checks within services
    // (e.g., "can this user export this specific script version?")
  ListOrgMembers(OrgRequest) → MembersResponse
}
```

**Events published:**
- `auth.user_created` — user ID, org ID
- `auth.user_deactivated` — user ID (triggers cascading access revocation in other services)

**Events consumed:**
- `auth.user_deactivated` → each service subscribes to revoke session data (Auth Service also publishes it)

---

### Audit Service

**Responsibility:** Immutable append-only audit log of every state-mutation event across the platform. Services do not write their own audit entries — the Audit Service derives them from domain events.

**Data ownership:**
- PostgreSQL schema `audit`: `audit_entries` (append-only; no UPDATE or DELETE ever issued)
- Alternatively: write-once object storage (S3 with Object Lock) for long-term compliance archives

**REST API (via HTTP Gateway, admin-only):**
- `GET /audit?resource_type=...&resource_id=...&from=...&to=...`

**Events consumed:** Every domain event from every service. The Audit Service is a universal subscriber.

**Why separate from each service's own logs?** Immutability is the key requirement. A compromised Script Service cannot delete its own audit entries if the audit service owns them. This separation is a compliance requirement for SOC 2 and ISO 27001.

---

## 3. Communication Architecture

### Three Planes

```
┌─────────────────────────────────────────────────────────────────────┐
│  CLIENT PLANE (React Web, Tauri On-Set)                              │
│                                                                      │
│  ┌────────────────────────────┐  ┌──────────────────────────────┐   │
│  │  HTTP Gateway              │  │  WebSocket (Collab) Gateway  │   │
│  │  GraphQL Federation        │  │  y-redis protocol            │   │
│  │  REST (upload, SSE, hooks) │  │  Presence + CRDT sync        │   │
│  └────────────┬───────────────┘  └──────────────┬───────────────┘   │
└───────────────┼──────────────────────────────────┼───────────────────┘
                │ gRPC (auth-injected headers)      │ WebSocket → Redis
┌───────────────▼──────────────────────────────────▼───────────────────┐
│  SERVICE PLANE (inside Istio mTLS mesh)                               │
│                                                                       │
│  Script ←gRPC→ Breakdown ←gRPC→ Scheduling ←gRPC→ Budget            │
│  Bible ←gRPC→ AI ←gRPC→ Search ←gRPC→ Continuity                    │
│  Import ←gRPC→ Script       Watermark ←gRPC→ Script                 │
│  Legal ←gRPC→ Supervisor    Auth ←gRPC→ Gateway                     │
│                                                                       │
│  ──────────────── NATS JetStream (event bus) ──────────────────────  │
│                                                                       │
│  Temporal Workers (one deployment per saga type)                      │
│    PublishSagaWorker, ImportSagaWorker, CallSheetWorker, etc.        │
└───────────────────────────────────────────────────────────────────────┘
                │ 
┌───────────────▼───────────────────────────────────────────────────────┐
│  DATA PLANE                                                            │
│  PostgreSQL (schema-per-service) | Neo4j | Elasticsearch | Redis | S3 │
└────────────────────────────────────────────────────────────────────────┘
```

### Communication Pattern Decision Table

| Scenario | Pattern | Technology |
|----------|---------|-----------|
| Client reads/writes (non-realtime) | GraphQL over HTTPS | Apollo Router → service subgraphs |
| Client file upload | REST multipart | HTTP Gateway → Import Service |
| Client AI streaming response | Server-Sent Events | HTTP Gateway → AI Service |
| Client real-time editing | WebSocket + CRDT ops | Collab Gateway → Redis Streams |
| Client notifications (in-app) | WebSocket push | Collab Gateway → Redis pub/sub |
| Service-to-service (request/response) | gRPC unary | Protobuf over HTTP/2, mTLS |
| Service-to-service (streaming) | gRPC server streaming | e.g., AI embedding generation |
| Async decoupled fan-out | Publish/subscribe | NATS JetStream |
| Long-running workflow coordination | Durable workflow | Temporal (activities = gRPC calls) |
| On-set → cloud data sync | Sync protocol | PowerSync over HTTPS |
| On-set photo upload | Presigned URL | Direct to S3 via HTTPS |
| External webhooks (NLE, ShotGrid) | REST | HTTP Gateway → Integration Hub |

---

## 4. API Gateway Design

### HTTP Gateway (Apollo Router)

The HTTP Gateway is built on **Apollo Router** running as a stateless, horizontally-scalable deployment. It handles:

1. **JWT validation** — Auth Service gRPC `ValidateToken` on every request; injects `X-User-ID`, `X-Org-ID`, `X-Roles`
2. **GraphQL query planning** — Federation query across subgraphs (script, breakdown, bible, continuity, scheduling, budget, legal, supervisor)
3. **REST routing** — forwarding non-GraphQL routes (upload, SSE, SCIM, SAML, audit)
4. **Rate limiting** — per-user, per-org, per-route; Redis-backed token buckets
5. **Request logging** — all requests (except CRDT WebSocket) logged for audit

```
HTTP Gateway Routes:

POST /graphql                  → Apollo Router (Federation)
POST /import/upload            → Import Service (REST, multipart)
GET  /ai/suggest               → AI Service (REST, SSE)
GET  /search                   → Search Service (REST)
GET  /export/{script_id}       → Script Service (REST, triggers Watermark inline)
GET  /auth/*                   → Auth Service (OIDC/OAuth flows)
POST /auth/saml/*              → Auth Service (SAML SP)
GET  /scim/v2/*                → Auth Service (SCIM)
POST /webhooks/nle             → Integration Hub
POST /webhooks/shotgrid        → Integration Hub
GET  /audit                    → Audit Service (admin-only)
GET  /health                   → Gateway health (unauthenticated)
```

### WebSocket (Collaboration) Gateway

Separate deployment. Handles:
1. **Initial JWT validation** — one-time check on WebSocket upgrade handshake; Auth Service gRPC
2. **User-to-document routing** — maps `ws://collab/{script_id}` to the correct Redis Stream
3. **Presence tracking** — maintains `presence:{script_id}` Redis hash (user ID → cursor position + name)
4. **Notification push** — subscribes to `notify:{user_id}` Redis pub/sub channel; forwards to connected client

```
WebSocket Gateway Routes:

ws:/collab/edit/{script_id}    → CRDT sync for document
ws:/collab/notify              → Notification push channel (per authenticated user)
```

### GraphQL Federation Subgraph Registry

| Subgraph | Service | Key Types |
|----------|---------|-----------|
| `script` | Script Service | `Project`, `Script`, `Episode`, `ASTNode`, `ScriptVersion` |
| `breakdown` | Breakdown Service | `BreakdownElement`, `BreakdownScene` |
| `bible` | Bible Service | `Character`, `Location`, `Fact`, `Arc`, `VoiceProfile` |
| `continuity` | Continuity Service | `StoryDay`, `ContinuityAssertion`, `ContinuityViolation` |
| `scheduling` | Scheduling Service | `ShootDay`, `ScheduledScene`, `DayOutOfDays` |
| `budget` | Budgeting Service | `BudgetVersion`, `BudgetLine` |
| `legal` | Legal Service | `RightsTag`, `NDARecord`, `LegalHold` |
| `supervisor` | Supervisor Service | `Setup`, `TakeLog`, `LineDeviation` |
| `auth` | Auth Service | `User`, `Organization`, `Role` |
| `ai` | AI Service | `LedgerEntry`, `ConsentRecord` |

**Federation key patterns:**

```graphql
# Script Service defines the canonical Scene
type Scene @key(fields: "id") {
  id: ID!
  type: ScriptNodeType!
  metadata: SceneMetadata!
}

# Breakdown Service extends Scene with breakdown data
extend type Scene @key(fields: "id") {
  id: ID! @external
  breakdown: [BreakdownElement!]!
}

# Continuity Service extends Scene with timeline data
extend type Scene @key(fields: "id") {
  id: ID! @external
  storyDay: StoryDay
  assertions: [ContinuityAssertion!]!
  openViolations: [ContinuityViolation!]!
}
```

This allows a single GraphQL query to get a scene with its breakdown and continuity data without the client knowing which service owns what.

---

## 5. Event Bus — NATS JetStream

### Stream Configuration

| Stream | Subjects | Retention | Replicas | Max Age |
|--------|----------|-----------|----------|---------|
| `SCRIPT` | `script.*`, `scene.*` | Limits | 3 | 30 days |
| `BIBLE` | `bible.*` | Limits | 3 | 30 days |
| `BREAKDOWN` | `breakdown.*` | Limits | 3 | 30 days |
| `PRODUCTION` | `schedule.*`, `budget.*`, `supervisor.*` | Limits | 3 | 30 days |
| `LEGAL` | `legal.*`, `watermark.*` | Work queue | 3 | 90 days |
| `AUTH` | `auth.*` | Work queue | 3 | 7 days |
| `NOTIFY` | `notify.*` | Work queue | 3 | 24 hours |
| `AUDIT` | `audit.*` | Limits | 3 | 7 years |

**Retention "Limits":** Message retained until all consumers have acknowledged OR max age reached. Allows replay for new consumers.

**Retention "Work queue":** Message deleted after single acknowledgement. For point-to-point delivery (Notification Service processing a notification once).

### Canonical Event Schema

Every event follows this envelope:

```typescript
interface DomainEvent<T = unknown> {
  id: string;           // UUID v7 (globally unique, used for idempotency)
  subject: string;      // NATS subject: "script.checkpointed"
  version: number;      // schema version (for forward compatibility)
  timestamp: string;    // ISO 8601
  source_service: string;
  actor_id: string;     // user UUID (or service account ID for system events)
  org_id: string;
  project_id: string | null;
  payload: T;
}
```

### Key Event Payloads

```typescript
// script.checkpointed
interface ScriptCheckpointedPayload {
  script_id: string;
  version: bigint;
  dirty_node_ids: string[];       // only the AST nodes whose content changed
  dirty_scene_ids: string[];      // scenes with any dirty child nodes
  checkpoint_type: 'auto' | 'manual' | 'publish';
}

// script.published
interface ScriptPublishedPayload {
  script_id: string;
  version_id: string;
  revision_color: RevisionColor;
  locked: boolean;
  page_count: number;
}

// scene.modified
interface SceneModifiedPayload {
  scene_id: string;
  script_id: string;
  changed_fields: string[];     // which metadata fields changed
  new_story_day_ref: string | null;
  new_character_refs: string[];
  new_location_ref: string | null;
}

// bible.fact_created
interface BibleFactCreatedPayload {
  fact_id: string;
  entity_id: string;
  entity_type: string;
  category: string;
  confidence: number;
  source_type: string;
  source_ref: string | null;
}

// continuity.violation_detected
interface ContinuityViolationPayload {
  violation_id: string;
  assertion_id: string;
  scene_id: string;
  severity: 'critical' | 'major' | 'minor' | 'informational';
  description: string;
}

// supervisor.take_logged
interface TakeLoggedPayload {
  take_id: string;
  scene_id: string;
  shoot_day_id: string;
  circled: boolean;
  printed: boolean;
  timecode_in: string;
  timecode_out: string;
}
```

### Consumer Groups

| Consumer | Subscribes to | Delivery Policy | Max Ack Pending |
|----------|--------------|----------------|----------------|
| `breakdown.indexer` | `script.checkpointed` | New from last ack | 100 |
| `search.script_indexer` | `script.checkpointed`, `script.published` | New from last ack | 50 |
| `search.bible_indexer` | `bible.fact_created`, `bible.conflict_detected` | New from last ack | 50 |
| `continuity.checker` | `scene.modified`, `script.checkpointed` | New from last ack | 25 |
| `notify.dispatcher` | All notification-triggering events | New from last ack | 500 |
| `audit.recorder` | All subjects (`>`) | New from last ack | 1000 |

**Idempotency:** Every consumer uses the event `id` (UUID v7) to deduplicate. If an event is redelivered (network failure, restart), the consumer checks if it has already processed this event ID before acting.

---

## 6. Temporal Orchestration Placement

### Architecture

```
Temporal Cloud (managed)
  ├── ScriptOS namespace
  │   ├── task queue: publish-saga
  │   ├── task queue: import-saga
  │   ├── task queue: call-sheet
  │   └── task queue: revision-distribution

Temporal Workers (in Kubernetes, inside Istio mesh)
  ├── publish-saga-worker  (1+ pods)
  │   └── Activities: lock_draft, validate_policy, extract_breakdown,
  │                   compute_schedule_delta, compute_budget_delta,
  │                   route_approvals, apply_operational_lock
  │
  ├── import-saga-worker  (2+ pods, CPU-intensive)
  │   └── Activities: detect_format, parse_to_ast, validate_ast,
  │                   generate_confidence, persist_ast, trigger_index
  │
  ├── call-sheet-worker  (1+ pods)
  │   └── Activities: pull_scenes, resolve_cast, resolve_crew,
  │                   fetch_weather, generate_pdf, await_approval, distribute
  │
  └── revision-dist-worker  (1+ pods)
      └── Activities: lock_revision, generate_diff, generate_colored_pages,
                      embed_watermarks, distribute_to_recipients
```

### Activity → Service Mapping

Every activity is an RPC call to a service inside the mesh. Temporal workers authenticate as service accounts (machine-to-machine JWT, injected at the mesh level).

```
Activity                    →  Service gRPC Call
────────────────────────────────────────────────────────────────────────
lock_draft                  →  ScriptService.LockDraft
unlock_draft                →  ScriptService.UnlockDraft
validate_policy             →  LegalService.ValidatePolicy
validate_canon              →  BibleService.CheckConflicts (all unresolved)
extract_breakdown           →  BreakdownService.ExtractBreakdown
compute_schedule_delta      →  SchedulingService.ComputeScheduleDelta
compute_budget_delta        →  BudgetService.ComputeBudgetDelta
route_approvals             →  (internal Temporal: signal workflow, await approval)
apply_operational_lock      →  ScriptService.ApplyOperationalLock
rollback_breakdown          →  BreakdownService.RollbackToVersion
rollback_schedule           →  SchedulingService.RollbackToVersion
rollback_budget             →  BudgetService.RollbackToVersion
detect_format               →  ImportService.DetectFormat
parse_to_ast                →  ImportService.ParseToAST
persist_ast                 →  ImportService.PersistAST
embed_watermark             →  WatermarkService.EmbedWatermark
distribute_watermarked      →  (email delivery via Notification Service gRPC)
```

### Workflow Timeout Policy

```yaml
publish_saga:
  execution_timeout: 72h      # full saga including approvals
  activity_schedule_to_close: 5m    # per automated step
  approval_activity_schedule_to_close: 48h  # for human approval steps
  retry_policy:
    initial_interval: 1s
    backoff_coefficient: 2.0
    maximum_attempts: 3
    non_retryable_errors:
      - PolicyViolationError      # policy failure is not transient
      - CanonConflictError        # unresolved canon conflict blocks publish

import_saga:
  execution_timeout: 4h       # OCR jobs can be slow
  activity_schedule_to_close: 30m
  retry_policy:
    maximum_attempts: 3
    non_retryable_errors:
      - UnsupportedFormatError
      - CorruptFileError
```

---

## 7. Auth & RBAC

### Token Flow

```
1. User authenticates via Auth Service (OIDC/SAML/email+password)
2. Auth Service issues JWT:
   {
     sub: "user_uuid",
     org_id: "org_uuid",
     roles: ["writer", "producer"],
     iat: ...,
     exp: ...,       // 1 hour
     refresh_token: "..."  // 30 days, rotated on use
   }
3. Client sends JWT in every HTTP/WebSocket request
4. HTTP Gateway validates JWT (Auth Service gRPC ValidateToken)
5. On success, Gateway injects:
   X-User-ID: user_uuid
   X-Org-ID: org_uuid
   X-Roles: writer,producer
6. Services read these headers — no JWT parsing in services
```

### RBAC Role Model

Roles are org-scoped. A user's roles are set per organization membership.

| Role | Description |
|------|-------------|
| `owner` | Org admin; all permissions |
| `producer` | Can publish drafts, approve workflows, manage scheduling/budget |
| `writer` | Can create/edit scripts; AI access; cannot publish or approve |
| `script_supervisor` | On-set module access; cannot edit scripts |
| `director` | Read script; add notes; approve takes |
| `department_head` | Breakdown and schedule read/write for their department |
| `editorial` | Read scripts and take logs; no edits |
| `legal` | Legal/rights module; cannot edit scripts |
| `exec` | Read-only access to all data; analytics |
| `service_account` | Machine-to-machine; used by Temporal workers and internal services |

**Project-level overrides:** A user's org role can be scoped per project. A writer in the org might be `read_only` on a specific project they're not assigned to.

### Permission Check Pattern

Fine-grained permissions (e.g., "can user X export script version Y?") are checked via Auth Service gRPC `CheckPermission`. This is called:
- By the HTTP Gateway for sensitive GraphQL mutations before routing
- By services for operations not covered by role alone (e.g., export requires NDA acceptance)

The `CheckPermission` call is cached in-process (Redis-backed) with a 60-second TTL to avoid N+1 gRPC calls on complex GraphQL queries.

---

## 8. Database Ownership Model

### PostgreSQL Schema Allocation

All schemas live in the same managed PostgreSQL cluster initially. Each service connects with its own database user that has access ONLY to its own schema.

```
postgres (cluster)
├── schema: script       ← Script Service user
├── schema: breakdown    ← Breakdown Service user
├── schema: scheduling   ← Scheduling Service user
├── schema: budget       ← Budgeting Service user
├── schema: continuity   ← Continuity Service user
├── schema: supervisor   ← Supervisor Service user (PowerSync reads this)
├── schema: ai           ← AI Service user
├── schema: import       ← Import Pipeline Service user
├── schema: watermark    ← Watermarking Service user
├── schema: legal        ← Legal Service user
├── schema: notify       ← Notification Service user
├── schema: auth         ← Auth Service user
├── schema: audit        ← Audit Service user (INSERT only)
└── schema: watermark    ← Watermarking Service user
```

**Rules enforced at the application layer:**
1. No Postgres FK constraints that cross schema boundaries
2. Cross-service references are stored as `UUID` (text/uuid column) — never a FK
3. No service reads another service's schema directly — all cross-service reads go through gRPC APIs
4. Migrations for each schema are owned by that service's deployment pipeline

**Read replicas:** A dedicated read replica is provisioned for:
- Search Service indexing pipeline (high read volume from `script` schema)
- Analytics queries (admin dashboard)
- Audit reporting

### Neo4j (Bible Service only)

Neo4j Aura managed instance. Bible Service is the only service with network access to Neo4j. All other services that need bible data go through Bible Service's gRPC API.

### Redis

Multiple Redis logical databases (db index) within a single Redis cluster:

| DB Index | Owner | Usage |
|----------|-------|-------|
| 0 | Collaboration Service | CRDT streams, presence |
| 1 | HTTP Gateway | Rate limiting (token buckets) |
| 2 | HTTP Gateway | Auth permission cache |
| 3 | AI Service | Semantic cache for prompt deduplication |
| 4 | Notification Service | In-app notification pub/sub |
| 5 | Session cache | JWT refresh token validation |

### Elasticsearch

Three indexes, owned by Search Service:

| Index | Content | Refresh Interval |
|-------|---------|-----------------|
| `scriptos-scenes` | AST node content snapshots, scene metadata, embeddings | 1 second |
| `scriptos-bible` | Bible entities, facts, voice profiles, embeddings | 1 second |
| `scriptos-production` | Take logs, production notes, supervisor notes | 5 seconds |

### S3 (Object Storage)

All services share one S3-compatible store with prefix-based ownership:

| Prefix | Owner | Contents |
|--------|-------|---------|
| `imports/` | Import Service | Original uploaded files (FDX, PDF, etc.) |
| `exports/` | Script Service | Generated PDF, FDX, Fountain exports |
| `snapshots/` | Script Service | `crdt_snapshot` binary backups |
| `continuity-photos/` | Supervisor Service | On-set continuity photos |
| `watermarked/` | Watermarking Service | Watermarked export cache |
| `reports/` | Various | Call sheets, turnover packages, budget exports |
| `thumbnails/` | Various | Image thumbnails for UI |

---

## 9. CRDT Collaboration Architecture

### Session Lifecycle

```
1. Client opens ws://collab/edit/{script_id} with JWT
2. Collab Gateway validates JWT → gets user info from Auth Service
3. Gateway subscribes to Redis Stream: crdt:{script_id}
4. Gateway loads last checkpoint from:
   a. Redis: latest ops in stream (since last Postgres checkpoint)
   b. Falls back to Script Service gRPC GetVersionSnapshot (CRDT bytea)
5. Gateway sends full state vector to client (sync protocol)
6. Client sends missing ops → Gateway → Redis Stream
7. Gateway fans out new ops to all other connected clients for this script_id

On first connection to a script with no Redis stream yet:
  → Script Service gRPC GetVersionSnapshot returns crdt_snapshot BYTEA
  → Gateway loads it into a temporary Loro/Yjs doc to extract state vector
  → Subsequent ops go via Redis
```

### Checkpoint Flow

```
Trigger: 5-minute idle OR explicit save OR publish request

1. Collaboration Service identifies all dirty leaf node IDs
   (nodes with CRDT ops since last checkpoint)
2. Calls Script Service gRPC CheckpointCRDT:
   - Input: script_id, { node_id → content_snapshot } map, structural changes
   - Script Service updates ast_nodes.content_snapshot in Postgres
   - Script Service re-derives scene metadata (int_ext, location, characters)
   - Script Service increments ast_nodes.version counters
   - Script Service publishes script.checkpointed event
3. Script Service gRPC returns checkpoint_version (bigint)
4. Collaboration Service records last_checkpoint_version in Redis
```

### Scaling Invariants

- **Any WebSocket server can handle any client** — no sticky sessions needed
- **Redis Stream is the only coordination point** — WS servers are pure forwarders
- **Worker processes are the only Postgres writers** — WS servers never write to Postgres directly
- **Presence data** is ephemeral Redis hash; lost on gateway restart (clients re-broadcast on reconnect)

### Structural Operation Coordination (Loro primary path)

With Loro MovableTree, scene moves are fully CRDT — no central coordinator needed. The WS gateway forwards the move op through the Redis stream and Loro resolves conflicts at each client.

With Yjs fallback, structural operations (scene moves, splits, reorders) require a lightweight serialization coordinator:
- The WebSocket server that receives the structural op claims a Redis lock for `struct_lock:{script_id}` (10-second TTL)
- It serializes the op as a Y.Map mutation and broadcasts via the stream
- Other WS servers forward it to their clients
- The lock ensures structural ops are totally ordered

---

## 10. On-Set Architecture (PowerSync Boundary)

### What PowerSync Syncs

PowerSync syncs between the cloud Postgres (Supervisor Service schema) and on-set SQLite. It does NOT sync Script Service data directly — the on-set device gets a read-only snapshot of relevant script data that the Supervisor Service has materialized.

**Cloud → Device (Sync Streams, read-only on device):**

| Table | Sync Rule | Condition |
|-------|-----------|-----------|
| `script.ast_nodes` | All scenes + elements for assigned episode | `script_id IN (?) AND deleted_at IS NULL` |
| `continuity.continuity_assertions` | Assertions for assigned project | `project_id = ?` |
| `breakdown.breakdown_elements` | Elements for assigned episode | `script_id = ?` |
| `scheduling.scheduled_scenes` | Scenes scheduled for current shoot week | `shoot_date BETWEEN ? AND ?` |
| `scheduling.shoot_days` | Current shoot week | `shoot_date BETWEEN ? AND ?` |

**Device → Cloud (write, queued offline):**

| Table | Owner | Conflict Resolution |
|-------|-------|---------------------|
| `supervisor.setups` | Device (creates new setups) | Add-wins (new setup never conflicts) |
| `supervisor.take_logs` | Device (appends takes) | Add-wins (new take never conflicts) |
| `supervisor.line_deviations` | Device | Add-wins |
| `continuity.continuity_assertions` | Device (status updates only) | LWW by updated_at |
| Continuity photo metadata | Device | Add-wins (new photo reference) |

**Note:** Continuity photos themselves are NOT synced via PowerSync. They are uploaded directly to S3 via a presigned URL when connectivity is available. Photo metadata (S3 key reference) is synced via PowerSync.

### On-Set Network Topology

```
Remote Location (no internet):
  iPad (Script Supervisor)  ─┐
  Tablet (1st AD)           ─┤── Local WiFi ──► Basecamp Laptop
  Director Phone            ─┘                   ├── PowerSync (local instance)
                                                  └── PostgreSQL (local)
                                                        │
                                                        │ (when internet available)
                                                        ▼
                                              Cloud PostgreSQL

Normal Location (internet available):
  iPad (Script Supervisor)  ──► Internet ──► PowerSync Cloud ──► Cloud PostgreSQL
```

### Tauri App Architecture

```
Tauri 2.x App (on-set device)
├── Rust backend
│   ├── SQLite access (via rusqlite + SQLCipher)
│   ├── PowerSync SDK (sync protocol, offline queue)
│   └── S3 presigned URL upload (for photos)
└── React frontend (WebView)
    ├── Script display (render AST from SQLite)
    ├── Take logging UI
    ├── Continuity photo capture
    └── Lined script view
```

The Rust backend handles all file system and database access. The React frontend communicates with the Rust backend via Tauri's `invoke` IPC. No direct network calls from the frontend — all network goes through the Rust backend layer.

---

## 11. Infrastructure Topology

### Kubernetes Cluster Layout

```
Kubernetes Cluster (per environment)
│
├── Namespace: scriptos-gateway
│   ├── deployment: http-gateway (Apollo Router, 3–10 pods, HPA)
│   └── deployment: ws-gateway (Collab Gateway, 5–20 pods, HPA)
│
├── Namespace: scriptos-core
│   ├── deployment: script-service (3 pods)
│   ├── deployment: bible-service (3 pods)
│   ├── deployment: breakdown-service (3 pods)
│   ├── deployment: scheduling-service (2 pods)
│   ├── deployment: budget-service (2 pods)
│   ├── deployment: continuity-service (3 pods)
│   ├── deployment: supervisor-service (2 pods)
│   ├── deployment: ai-service (3 pods)
│   ├── deployment: import-service (2 pods, CPU-intensive)
│   ├── deployment: search-service (2 pods)
│   ├── deployment: watermark-service (2 pods)
│   ├── deployment: legal-service (2 pods)
│   ├── deployment: notification-service (3 pods)
│   ├── deployment: auth-service (3 pods)
│   └── deployment: audit-service (2 pods)
│
├── Namespace: scriptos-workers
│   ├── deployment: publish-saga-worker (2 pods)
│   ├── deployment: import-saga-worker (3 pods, CPU-intensive for OCR/parsing)
│   ├── deployment: call-sheet-worker (1 pod)
│   └── deployment: revision-dist-worker (1 pod)
│
├── Namespace: scriptos-infra
│   ├── statefulset: nats (3-node JetStream cluster)
│   └── (Redis, Postgres, Neo4j, ES are managed external services)
│
└── Namespace: scriptos-monitoring
    ├── prometheus
    ├── grafana
    ├── jaeger (or tempo)
    └── loki
```

### Managed External Services

| Service | Managed Option | VPC Integration |
|---------|---------------|----------------|
| PostgreSQL | Cloud SQL (GCP) / RDS (AWS) Multi-AZ | Private IP, same VPC |
| Redis | Upstash (serverless) / ElastiCache | Private endpoint or VPC |
| Neo4j | Neo4j Aura Professional | Private Link / VPN |
| Elasticsearch | Elastic Cloud | VPC traffic filtering |
| Temporal | Temporal Cloud | gRPC over internet (TLS) |
| S3 | Cloud native (GCS / S3) | VPC endpoint / private access |

All managed services are reachable only via private endpoints or VPC peering. No managed service is exposed to the public internet.

### Multi-Region Topology (Production SaaS)

```
Region 1 (Primary — us-east or eu-west):
  ├── All services (full deployment)
  ├── PostgreSQL primary (all writes)
  ├── Redis primary (CRDT streams + cache)
  ├── Neo4j (single region — consistency over latency)
  ├── Elasticsearch (primary shards)
  └── WebSocket Gateway (CRDT sync)

Region 2 (Secondary):
  ├── HTTP Gateway (GraphQL reads from Postgres read replica)
  ├── WebSocket Gateway (CRDT sync — Redis Streams are global via Upstash or Redis Cluster)
  ├── PostgreSQL read replica (for read-heavy GraphQL queries)
  ├── Elasticsearch replica shards (search reads)
  └── AI Service (reduces latency for writers in this region)
  └── (No Temporal workers — single region for saga consistency)
```

**CRDT cross-region:** Redis Streams used for CRDT fan-out are global (Upstash Global, or Redis Cluster with cross-region replication). A writer in Region 2 connects to the Region 2 WebSocket Gateway, which reads/writes the same Redis Stream as Region 1 clients editing the same document. CRDT convergence handles any ordering differences.

**Write consistency:** All Postgres writes go to the primary in Region 1. GraphQL mutations are routed to Region 1. GraphQL queries can read from the nearest replica with a 1-second propagation lag (acceptable for most reads; the Script Service serves real-time scene content from the CRDT layer, not from Postgres).

### Horizontal Pod Autoscaling Triggers

| Deployment | Scale On | Min | Max |
|------------|----------|-----|-----|
| HTTP Gateway | CPU 60% | 3 | 20 |
| WebSocket Gateway | Active connections per pod > 5,000 | 5 | 50 |
| Import Service | CPU 70% (OCR is CPU-bound) | 2 | 10 |
| AI Service | Pending request queue depth | 3 | 15 |
| Script Service | CPU 60% | 3 | 10 |
| Import Saga Workers | Temporal task queue depth | 2 | 8 |

---

## 12. Observability

### Distributed Tracing

Every request carries an OpenTelemetry trace context. The trace propagates across gRPC calls, NATS message processing, and Temporal activities.

**Critical traces to instrument:**
- Full GraphQL request → subgraph → service gRPC → Postgres query
- CRDT op → Redis Stream → checkpoint → Postgres write
- Temporal activity chain for publish saga
- AI request → context assembly (Bible gRPC + Script gRPC) → model API → ledger write

**Trace sampling:** 100% for errors; 5% for success (adjustable per environment).

### Key Metrics

| Metric | Service | Alert Threshold |
|--------|---------|----------------|
| `http_request_duration_p99` | HTTP Gateway | > 500ms |
| `ws_connection_count` | Collab Gateway | > 80% of max |
| `crdt_checkpoint_lag_seconds` | Collab Service | > 10 minutes |
| `breakdown_extraction_duration_p99` | Breakdown | > 60 seconds |
| `ai_request_duration_p99` | AI Service | > 30 seconds |
| `nats_consumer_lag` | NATS | > 1,000 messages |
| `temporal_workflow_failures` | Temporal | Any non-zero |
| `pg_replication_lag_seconds` | PostgreSQL | > 5 seconds |
| `es_indexing_lag_seconds` | Search Service | > 30 seconds |
| `powersync_sync_errors` | Supervisor | Any |

### Logging Standards

Structured JSON logs (not line-based). Every log entry includes:
- `trace_id` — OpenTelemetry trace ID
- `span_id`
- `service` — service name
- `level` — info/warn/error
- `user_id` — from `X-User-ID` header (if authenticated)
- `org_id`
- `project_id` (if available)
- `script_id` (if available)
- `message`

Log aggregation: Loki (Grafana stack) or OpenSearch.

### Alerting Runbook Owners

| Alert | Primary | Secondary |
|-------|---------|-----------|
| Publish saga failure | Backend on-call | Product |
| CRDT checkpoint lag | Infrastructure | Backend |
| AI service degraded | AI team | Backend |
| PowerSync sync failure | Backend | On-set support |
| Auth service down | Infrastructure | All |
| Neo4j unreachable | Infrastructure | Backend (Bible) |

---

## 13. Repository Structure

See **ADR-017** for the full rationale. Summary: monorepo with pnpm workspaces + Turborepo for TypeScript, Cargo workspace for Rust. Services are deployed as separate container images — monorepo is a development-time concept only.

```
scriptos/
│
├── packages/                        ← shared libraries, imported by services and apps
│   ├── ast/                         ← ASTNode, ScriptNodeType, RevisionColor, BreakdownTag
│   ├── proto/                       ← .proto files + generated TypeScript gRPC stubs
│   │   ├── script.proto
│   │   ├── bible.proto
│   │   ├── breakdown.proto
│   │   └── ... (one per service)
│   ├── events/                      ← NATS DomainEvent<T> envelope + all payload types
│   ├── workflows/                   ← Temporal workflow definitions + activity interfaces
│   ├── db/                          ← Postgres migration tooling (Atlas/Flyway config per schema)
│   └── common/                      ← auth header utils, OpenTelemetry setup, logger factory
│
├── services/                        ← microservices (each has own Dockerfile + package.json)
│   ├── script/
│   ├── collaboration/
│   ├── bible/
│   ├── breakdown/
│   ├── scheduling/
│   ├── budget/
│   ├── continuity/
│   ├── supervisor/
│   ├── ai/
│   ├── import/
│   ├── search/
│   ├── watermark/
│   ├── legal/
│   ├── notification/
│   └── auth/
│
├── workers/                         ← Temporal workers (separate Kubernetes deployments)
│   ├── publish-saga/
│   ├── import-saga/
│   ├── call-sheet/
│   └── revision-dist/
│
├── apps/
│   ├── web/                         ← React web app (imports from packages/ast, packages/common)
│   ├── onset/                       ← Tauri on-set app
│   │   ├── src-tauri/               ← Rust backend (Cargo workspace member)
│   │   └── src/                     ← React frontend (shares components with apps/web)
│   └── gateway/                     ← Apollo Router config, supergraph schema composition
│
├── infra/
│   ├── k8s/                         ← Kubernetes manifests per service
│   ├── terraform/                   ← Cloud infrastructure (VPC, managed services)
│   └── helm/                        ← Helm charts for Kubernetes deployment
│
├── Cargo.toml                       ← Cargo workspace root
├── package.json                     ← pnpm workspace root
├── pnpm-workspace.yaml              ← lists all packages/services/apps/workers
└── turbo.json                       ← Turborepo pipeline (build, test, lint, docker)
```

### Dependency Direction Rule

```
apps/*      →  can import from packages/*
services/*  →  can import from packages/*
workers/*   →  can import from packages/workflows, packages/proto, packages/common
services/*  ✗  must NOT import from other services/*
apps/*      ✗  must NOT import from services/* directly
```

This is enforced with an ESLint `import/no-restricted-paths` rule in CI. A service that needs another service's data calls its gRPC API — it does not import its source code.

### Turborepo Pipeline

```json
// turbo.json
{
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**"]
    },
    "proto:gen": {
      "inputs": ["../../packages/proto/**/*.proto"],
      "outputs": ["src/generated/**"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "docker:build": {
      "dependsOn": ["build", "test"],
      "cache": false
    },
    "lint": {}
  }
}
```

`dependsOn: ["^build"]` means a service only builds after all its `packages/*` dependencies have built. `proto:gen` is cached — proto stubs are only regenerated when `.proto` files change.

### CI Affected Graph

On a PR that touches only `services/breakdown/`:
- Turborepo computes the affected graph: `breakdown` depends on `packages/ast`, `packages/proto`, `packages/events`
- If none of those packages changed, only `services/breakdown/` is built, tested, and Docker-built
- All other services hit the remote build cache and are skipped

On a PR that touches `packages/ast/`:
- Every service and app that imports `packages/ast` is rebuilt and tested (caught by Turborepo affected graph)
- This is the intended behaviour — a type change in AST propagates its impact to all consumers immediately

---

## 14. Service Dependency Map

Reading as: **Row → depends on → Column**. ✅ = synchronous gRPC call. 📨 = event subscription. 🔄 = Temporal activity.

|  | Script | Bible | Breakdown | Scheduling | Budget | Continuity | AI | Import | Supervisor | Auth | Watermark | Legal | Search | Notify |
|--|--------|-------|-----------|-----------|--------|-----------|-----|--------|-----------|------|-----------|-------|--------|-------|
| **HTTP Gateway** | — | — | — | — | — | — | — | — | — | ✅ | — | — | — | — |
| **Collab Gateway** | ✅ | — | — | — | — | — | — | — | — | ✅ | — | — | — | — |
| **Script** | — | — | — | — | — | — | — | — | — | — | ✅ | — | — | — |
| **Breakdown** | 📨 | ✅ | — | — | — | — | — | — | — | — | — | — | — | — |
| **Scheduling** | — | — | 📨 | — | — | — | — | — | — | — | — | — | — | — |
| **Budget** | — | — | — | 📨 | — | — | — | — | — | — | — | — | — | — |
| **Continuity** | 📨 | — | — | — | — | — | ✅ | — | — | — | — | — | — | — |
| **AI** | ✅ | ✅ | — | — | — | — | — | — | — | — | — | — | — | — |
| **Search** | 📨 | 📨 | — | — | — | — | ✅ | — | — | — | — | — | — | — |
| **Import** | ✅ | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **Notification** | 📨 | 📨 | 📨 | 📨 | — | 📨 | — | 📨 | 📨 | — | — | — | — | — |
| **Audit** | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | 📨 | — | 📨 |
| **Temporal Workers** | 🔄 | 🔄 | 🔄 | 🔄 | 🔄 | — | — | 🔄 | — | — | 🔄 | 🔄 | — | — |

**Critical path for publish saga** (must all be healthy to publish):
`Script → Legal → Bible → Breakdown → Scheduling → Budget → Script`

**Single points of failure to design for:**
- Auth Service: every request fails if Auth is down. Requires: multi-pod deployment, circuit breaker in gateway (serve cached permission results for 60s during Auth degradation)
- Redis: CRDT sync fails, notification push fails. Requires: Redis Sentinel or Cluster with auto-failover
- PostgreSQL primary: all writes fail. Requires: Multi-AZ with automatic failover (< 30 second RTO)
- Temporal Cloud: all sagas pause (workflows preserve state, resume automatically when connectivity returns). Acceptable.

---

## 15. Open Items for Deep-Dives

These are not design gaps — the system design is complete. These are specific technical questions that belong in the per-module deep-dive documents (Step 3).

### CRDT Deep-Dive (wiki/04)
- Loro ProseMirror binding: build in-house or contribute upstream?
- Awareness/cursor protocol design for Loro (y-protocols equivalent)
- CRDT compaction policy: frequency and trigger
- Maximum validated concurrent editor count

### AI Governance Deep-Dive (wiki/05)
- Prompt architecture for Bible + Timeline + Draft context assembly
- Embedding model selection (Voyage-3 vs text-embedding-3-large vs self-hosted)
- Voice registry: how many reference scenes per character for few-shot effectiveness
- C2PA integration point: PDF export, FDX export, or both

### Offline Sync Deep-Dive (wiki/06)
- PowerSync self-hosted licensing for basecamp deployments
- Photo sync strategy: on-device compression vs. upload originals
- SQLCipher key management: device keychain, user passphrase, or org-managed
- Tauri 2.x iOS maturity evaluation (go/no-go decision for iPad release)

### Orchestration Deep-Dive (wiki/07)
- Approval routing: configurable per-org workflows vs hardcoded approval chains
- Escalation policy: auto-approve after timeout or escalate to next level
- Multi-department approval: sequential vs parallel for publish saga step 6
- Temporal Cloud cost projections at 100/1000 active productions

### Breakdown & Scheduling Deep-Dive (wiki/08)
- NLP inference: co-deployed sidecar vs shared ML inference service
- Stripboard scheduling solver: build in-house vs integrate solver library
- Budget template library: which templates to ship at launch

### Search Deep-Dive (wiki/10)
- Embedding model: Voyage-3 vs OpenAI vs self-hosted (cost/quality tradeoff)
- Bible fact index strategy: separate from scene index or inline
- Character name fuzzy matching (JAKE/Jake/J./JACOB)

### Security Deep-Dive (wiki/11)
- Watermarking technology: build in-house vs Digimarc license
- SOC 2 Type II timeline and scope
- Data residency for EU: whether multi-region or EU-only single region

### Infrastructure Deep-Dive (wiki/12)
- y-redis commercial license cost confirmation
- Kubernetes provider decision: GKE vs EKS vs AKS
- Database migration tooling: Flyway vs Atlas vs custom
- Object storage: S3 vs Cloudflare R2 (cost/egress tradeoffs)

---

## Appendix: ADR Updates

The following ADRs should be added to `wiki/14-adr.md` based on decisions in this document:

```
ADR-009: Event Bus — NATS JetStream
ADR-010: Client API — GraphQL Federation (Apollo Router)
ADR-011: Internal Communication — gRPC
ADR-012: Auth Delegation — Centralized JWT Validation at HTTP Gateway
ADR-013: Database Isolation — Schema-per-Service on Shared Cluster
ADR-014: Temporal Deployment — Temporal Cloud (Managed)
ADR-015: Two-Gateway Architecture — HTTP (Apollo) + WebSocket (Collaboration)
ADR-016: On-Set Sync Boundary — PowerSync Manages All Supervisor Data
ADR-017: Repository Strategy — Monorepo (pnpm workspaces + Turborepo + Cargo workspace)
```
