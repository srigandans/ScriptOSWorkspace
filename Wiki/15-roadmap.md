# 15 — Implementation Roadmap

> **Status:** Step 4 — Sprint-level plan. All architectural decisions closed in wiki/14. All technical deep-dives complete (wiki/04–09). This document is the engineering execution plan.

---

## Team Composition Assumption

| Role | Count | Owns |
|------|-------|------|
| Backend engineer | 5–7 | Services, gRPC, Temporal workers, Postgres DDL, NATS |
| Frontend engineer | 2–3 | React web app, TipTap editor, Tauri on-set app |
| DevOps / Platform | 1 | GKE, CI/CD, Redis, NATS, Atlas migrations, observability |
| ML / AI engineer | 1 | AI service, model routing, embeddings, vLLM (enterprise) |

**Sprint cadence:** 2 weeks. **Total sprints:** 18 (36 weeks to GA).

Timelines assume 8 engineers (5 BE + 2 FE + 1 DevOps). A 12-engineer team compresses Phases 2–4 by ~25%. A 5-engineer team extends each phase by 50–75%.

---

## Service Bring-Up Dependency Order

The table below is the authoritative build sequence. A service may not be started until all its declared dependencies are marked **done** (i.e., passing integration tests and deployed to staging).

```
Tier 0 (must exist before any service)
  Auth Service
  PostgreSQL cluster (GKE managed + Atlas migrations)
  Redis
  NATS JetStream
  S3 buckets + IAM

Tier 1 (no inter-service dependencies beyond Tier 0)
  Script Service
  Import Service

Tier 2 (depends on Script)
  Collaboration Service   (depends on Script + Redis)
  Bible Service           (depends on Script + Neo4j)
  AI Service              (depends on Script + Bible + Elasticsearch)

Tier 3 (depends on Script + Collaboration + Bible)
  Breakdown Service       (depends on Script + Bible)
  Search Service          (depends on Script + Bible + Elasticsearch)
  Watermark Service       (depends on Script)

Tier 4 (depends on Breakdown)
  Scheduling Service      (depends on Breakdown)
  Budget Service          (depends on Scheduling)
  Continuity Service      (depends on Bible + Script)

Tier 5 (depends on Scheduling + Temporal)
  Supervisor Service      (depends on Script + Continuity + PowerSync)
  Legal Service           (depends on Script + Auth)
  Notification Service    (depends on Auth + NATS)

Tier 6 (depends on all above)
  Temporal Workers:       publish-saga, import-saga, call-sheet, revision-dist
  Editorial (post-prod)   (depends on Script + Supervisor)
```

---

## Sprint 0 — Infrastructure Foundation (Weeks 1–2)

**Goal:** Every engineer has a working local dev environment. CI/CD passes on an empty repo. GKE staging cluster running.

### Backend / Platform

| Task | Owner | Done When |
|------|-------|-----------|
| Create monorepo: pnpm workspaces + Turborepo + Cargo workspace | DevOps | `pnpm build` completes with no errors across all packages |
| GitHub Actions CI pipeline | DevOps | PR → lint + typecheck + test (empty suites pass) → Docker build |
| GKE Autopilot cluster (staging) | DevOps | `kubectl get nodes` returns 3+ nodes |
| PostgreSQL managed instance (Cloud SQL) | DevOps | Instance running; `psql` connection from cluster confirmed |
| Redis (Memorystore or self-hosted) | DevOps | `redis-cli ping` from cluster returns PONG |
| NATS JetStream | DevOps | JetStream cluster running; `nats stream ls` works |
| S3 buckets: `scriptos-media`, `scriptos-exports`, `scriptos-backups` | DevOps | Buckets exist; IAM role allows read/write from cluster |
| Neo4j Aura instance | DevOps | Neo4j Aura provisioned; connection string in Secrets Manager |
| Elasticsearch 8.9+ | DevOps | Cluster running; health check returns green |
| Atlas CLI setup; `packages/db` Atlas project init | BE-1 | `atlas schema apply` runs without error on empty DB |
| Istio service mesh installed on GKE | DevOps | mTLS enforced between namespaces; `istioctl analyze` clean |
| OpenTelemetry collector + Grafana + Prometheus | DevOps | Traces visible for a hello-world service |
| Temporal Cloud namespace provisioned | DevOps | Namespace `scriptos.{id}` active; TLS certs in Secrets Manager |
| Docker Compose dev env (`compose.yml`) | DevOps | `docker compose up` starts Postgres + Redis + NATS + Neo4j locally |
| Secrets management (GCP Secret Manager) | DevOps | All secrets mounted as env vars via External Secrets Operator |

### Sprint 0 Definition of Done

- [ ] `docker compose up` starts full local backend stack in < 2 minutes
- [ ] CI green on `main` branch (lint, typecheck, build)
- [ ] GKE staging cluster with Istio, OpenTelemetry, Prometheus running
- [ ] All Tier 0 dependencies operational in staging
- [ ] Every engineer ran `pnpm install` and `docker compose up` successfully on their machine
- [ ] Atlas migrations run cleanly against a fresh Postgres instance

---

## Sprint 1 — Auth Service + Shared Packages (Weeks 3–4)

**Goal:** Authentication working end-to-end. Shared TypeScript packages established. First service (Auth) deployed to staging.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/ast` — all 19 ASTNode types + RevisionColor enum | BE-1 | `tsc --noEmit` passes; types imported by `services/script` |
| `packages/events` — NATS DomainEvent envelope + all payload types | BE-2 | Event types compile; used by script + breakdown stubs |
| `packages/proto` — first .proto files: auth, script, breakdown stubs | BE-1 | `buf generate` produces TypeScript stubs; no compile errors |
| `packages/common` — logger (Pino), OpenTelemetry init, auth-header utils | BE-3 | Used by Auth Service; traces appear in Grafana |
| `packages/db` — Atlas schema for `auth.*` tables | BE-2 | `atlas schema apply` creates tables on Postgres |
| Auth Service: OIDC/SAML provider (Google, Okta) | BE-3 | Login via Google returns JWT; token validates |
| Auth Service: JWT signing (RS256, 15-min expiry, refresh token 30 days) | BE-3 | `/auth/token` and `/auth/refresh` endpoints working |
| Auth Service: RBAC — roles: `owner`, `admin`, `writer`, `producer`, `supervisor`, `editor`, `viewer` | BE-3 | Role stored in JWT claims; X-Roles header injected by gateway |
| Auth Service: SCIM provisioning endpoint (for enterprise SSO) | BE-4 | Stub only — returns 501 for now; spec documented |
| Auth Service: GraphQL subgraph (`User`, `Organization` types) | BE-3 | `user { id email roles }` query works through Apollo Router |
| HTTP Gateway: Apollo Router config + Auth Service supergraph entry | BE-1 | Apollo Router running; Auth subgraph federated |
| HTTP Gateway: JWT validation middleware | BE-1 | Request without valid JWT → 401; with valid JWT → headers injected |
| WebSocket Gateway: stub server (no CRDT yet) | BE-2 | WS connection accepted; JWT validated on handshake |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| React app scaffold (`apps/web`) — Vite + React 18 + Tailwind + React Router | FE-1 | `pnpm dev` serves app at localhost:3000 |
| Login page (Google OAuth flow) | FE-1 | Login → redirect → JWT stored in httpOnly cookie |
| Auth context + protected route wrapper | FE-1 | Unauthenticated → redirect to /login; authenticated → renders app |

### Sprint 1 Definition of Done

- [ ] Login via Google OAuth works end-to-end (browser → Auth Service → JWT)
- [ ] Apollo Router running in staging; Auth subgraph returns `{ me { id email } }`
- [ ] JWT validated at HTTP Gateway; X-User-ID header injected
- [ ] All `packages/*` compile with zero TypeScript errors
- [ ] Auth Service Docker image built and deployed to staging

---

## Sprint 2 — Script Service Core + AST Schema (Weeks 5–6)

**Goal:** Script AST schema in Postgres. Script Service running. Create/read project and script operations working.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Atlas schema for `script.*` tables (`projects`, `scripts`, `ast_nodes`, `script_versions`) | BE-1 | Tables exist in staging; indexes created |
| Script Service: project CRUD (create, read, list, archive) | BE-2 | `createProject` mutation; `projects` query paginated |
| Script Service: script CRUD within project | BE-2 | `createScript`; `script(id)` query returns metadata |
| Script Service: AST node upsert (batch) | BE-1 | `upsertASTNodes` gRPC method — inserts/updates `ast_nodes` rows |
| Script Service: scene metadata read (heading, page count, status) | BE-2 | `getSceneSnapshot` gRPC — returns heading + metadata for a scene |
| Script Service: version snapshot write | BE-1 | `createVersion` creates a row in `script_versions` with CRDT snapshot |
| Script Service: lock/unlock draft (for publish saga) | BE-2 | `lockDraft` / `unlockDraft` gRPC methods |
| Script Service: GraphQL subgraph — Project, Script, ASTNode types | BE-2 | `script(id) { scenes { sceneNumber location } }` works |
| Script Service: NATS publisher — `script.checkpointed` event | BE-1 | Event published after successful checkpoint; visible in NATS dashboard |
| Proto definitions: ScriptService complete | BE-1 | `packages/proto/script.proto` complete; buf generate succeeds |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Project list page | FE-1 | Shows projects for authenticated user |
| Create project modal | FE-1 | Creates project via GraphQL mutation; appears in list |
| Script list within project | FE-2 | Shows scripts in a project |
| Create script modal | FE-2 | Creates script; navigates to editor (empty) |

### Sprint 2 Definition of Done

- [ ] `createProject` → `createScript` → `script { scenes }` GraphQL flow working in staging
- [ ] `ast_nodes` table seeded with 3 test scenes; readable via gRPC `GetSceneSnapshot`
- [ ] Script Service Docker image deployed; health check passing
- [ ] Proto definitions for ScriptService complete and checked in
- [ ] Atlas migration for `script.*` schema applied to staging DB

---

## Sprint 3 — Script Editor + CRDT Collaboration (Weeks 7–8)

**Goal:** Real-time collaborative screenplay editor working. Two users can co-edit a script simultaneously. Checkpoint writes to Postgres.

**This is the highest-risk sprint — Loro POC gates the entire CRDT strategy.**

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Collaboration Service: Hocuspocus server setup (startup CRDT server) | BE-1 | WS connection to collab service; Loro doc syncs between two clients |
| Collaboration Service: `loadDocState()` — Redis → Postgres fallback chain | BE-1 | Doc loads from Redis on hot path; falls back to `script_versions` snapshot |
| Collaboration Service: `checkpoint()` — CRDT → `ast_nodes` write + NATS event | BE-2 | After idle timeout, ast_nodes updated; `script.checkpointed` published |
| Collaboration Service: awareness protocol (cursor + presence) | BE-2 | Two clients see each other's cursors with correct colors |
| Collaboration Service: struct ops — scene move with Redis lock | BE-3 | `handleSceneMoveOp` acquires lock; moves scene in Loro tree; releases |
| Collaboration Service: scene split + merge (atomic MovableTree ops) | BE-3 | Split scene → two scenes with correct headings; merge → one scene |
| CRDT POC gate (Week 6 decision): Loro passes 5 criteria → proceed; else Yjs fallback | BE-1 | Pass criteria documented in `Wiki/04-crdt-collaboration.md §Loro POC` |
| WebSocket Gateway: route collab connections to Collaboration Service | BE-1 | WS handshake → JWT validated → routed to correct script doc |
| `packages/proto`: CollaborationService proto complete | BE-1 | `CheckpointCRDT` gRPC callable from Script Service |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| TipTap editor scaffold with screenplay schema | FE-1 | Editor renders with scene heading, action, dialogue blocks |
| Loro ↔ ProseMirror binding (`LoroPMBinding` class) | FE-1 | Typing in editor → Loro op; remote op → PM transaction |
| Auto-formatting: Tab key cycles element types (scene heading → action → character → dialogue) | FE-2 | Tab cycles types per WGA format spec |
| Collaborative presence: show remote cursors with name labels | FE-2 | Two browser tabs show mutual cursor positions |
| Screenplay formatting: correct margins, fonts (Courier 12pt), page breaks | FE-1 | Visual output matches Final Draft formatting spec |
| Scene panel: left sidebar listing scenes with status | FE-2 | Sidebar updates as scenes are added in editor |

### Sprint 3 Definition of Done

- [ ] **CRDT POC gate passed:** Two clients co-edit a scene; changes converge; both see identical text after network partition + reconnect
- [ ] `checkpoint()` writes `ast_nodes` rows readable via Script Service gRPC
- [ ] Scene move preserves content; convergence within 200ms of op receipt
- [ ] Awareness: cursor positions visible across two browser tabs
- [ ] Collaboration Service deployed to staging; WS connections stable for 30 minutes under load
- [ ] **Internal dogfood target (Week 8): team writes scripts in ScriptOS daily**

---

## Sprint 4 — Import Pipeline (Weeks 9–10)

**Goal:** FDX and Fountain scripts importable. Import saga running in Temporal. Imported script editable in the CRDT editor.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Import Service: format detection (FDX, Fountain, PDF, Scanned PDF) | BE-4 | 5 test files auto-detected correctly |
| Import Service: FDX parser (XML → AST) | BE-4 | 10 reference FDX files parse with confidence ≥ 0.95 |
| Import Service: Fountain parser | BE-4 | 5 reference Fountain files parse with confidence ≥ 0.90 |
| Import Service: PDF text extraction (pdftotext/PyMuPDF) + positional heuristics | BE-5 | 3 digital PDF screenplays parse with correct element types |
| Import Service: AST normalization + confidence scoring | BE-4 | Every imported element has confidence score; warnings generated |
| Import Service: source provenance record | BE-4 | `ImportProvenance` written to DB; S3 archive of original file |
| Import Saga (Temporal worker): detect → parse → validate → user review → persist → index | BE-1 | End-to-end saga completes for FDX input in < 60s |
| Import Saga: batch import (zip of FDX/Fountain files) | BE-4 | 5 FDX files imported as 5 episodes; episode numbers inferred from filenames |
| Round-trip CI tests: import FDX → export FDX → import → assert structural equality | BE-4 | 10 reference scripts pass round-trip test in CI |
| Revision color preservation from FDX `RevisionChange` attributes | BE-4 | Imported AST nodes carry `revision_color` from source FDX |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Import modal: drag-and-drop or file picker (FDX, Fountain, PDF) | FE-2 | File upload triggers import saga; progress indicator shown |
| Import review UI: confidence warnings displayed per element | FE-2 | Low-confidence elements highlighted; accept/reject per element |
| Import progress: Temporal workflow status polled via GraphQL | FE-2 | Status updates: detecting → parsing → reviewing → persisting → done |
| Batch import: zip file upload → episode list preview → confirm → import | FE-2 | 3 episode zip imports correctly |

### Sprint 4 Definition of Done

- [ ] 10 reference scripts pass automated round-trip CI test (FDX → AST → FDX)
- [ ] FDX import end-to-end: upload → saga completes → script editable in CRDT editor
- [ ] Import provenance record written; original file archived in S3
- [ ] Batch import of 3-episode zip completes < 3 minutes
- [ ] Import Service deployed to staging

---

## Sprint 5 — Series Bible Graph (Weeks 11–12)

**Goal:** Bible Service operational. Characters, locations, lore, facts stored in Neo4j. Bible UI functional. Bible ↔ Script linking working.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Neo4j Cypher constraints + vector index (wiki/03 schema) | BE-2 | `SHOW CONSTRAINTS` lists all 15+ constraints; vector index exists |
| Bible Service: character CRUD (create, read, update, link to scenes) | BE-2 | Character node created; linked to scene via `APPEARS_IN` |
| Bible Service: location CRUD + linkage | BE-2 | Location linked to scene via `SET_IN` |
| Bible Service: fact creation with provenance (source scene, confidence, supersedure) | BE-5 | Fact has `source_scene_id`; supersedure chain traversable |
| Bible Service: world rule CRUD | BE-2 | World rules queryable; used by AI context assembly |
| Bible Service: canon consistency check (`validateScriptConsistency` gRPC) | BE-5 | Returns conflicts when a new fact contradicts an established fact |
| Bible Service: character voice profile CRUD | BE-5 | Voice profile stored; `forbidden_words` array; `reference_scenes` array |
| Bible Service: GraphQL subgraph — Character, Location, Fact, VoiceProfile types | BE-2 | `character(id) { facts { factText } voiceProfile { forbiddenWords } }` works |
| Bible Service: `getWorldRules`, `getCharacterFacts`, `getVoiceProfiles` gRPC (for AI context) | BE-5 | AI Service can call these for context assembly |
| Script Service: extract character + location names from scene headings on checkpoint | BE-1 | `script.checkpointed` → Bible Service extracts mentions → creates stubs |
| Proto definitions: BibleService complete | BE-2 | `packages/proto/bible.proto` complete |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Bible panel: tabbed view (Characters, Locations, Props, Lore) | FE-1 | CRUD for each entity type |
| Character detail page: facts list, voice profile editor, reference scene picker | FE-1 | Voice profile editable; reference scenes selectable from script |
| Location detail page: facts, linked scenes | FE-2 | Location shows all scenes where it appears |
| Bible ↔ editor integration: character names highlighted in editor; click → Bible panel | FE-1 | Clicking "WALTER" in dialogue → opens Walter's bible entry |
| Conflict alert banner: canon violations surfaced in editor | FE-2 | Red banner: "New fact contradicts established canon — review" |

### Sprint 5 Definition of Done

- [ ] Character → Facts → Voice Profile → Reference Scenes round-trip in Neo4j
- [ ] Canon consistency check returns conflicts for a known contradictory test case
- [ ] Bible subgraph federated into Apollo Router
- [ ] Character names auto-extracted from script on checkpoint
- [ ] Bible panel functional in web app; facts editable

---

## Sprint 6 — Breakdown Service + AI Service (Weeks 13–14)

**Goal:** Breakdown elements auto-detected from AST. AI dialogue suggestions working end-to-end with WGA-compliant ledger.

**Alpha milestone target (Week 14): 10 external writers using editor + bible.**

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Breakdown Service: NLP element detection (characters, locations, props, wardrobe, VFX, stunts) | BE-5 | 5 test scenes auto-tagged with ≥ 80% F1 score vs hand-tagged ground truth |
| Breakdown Service: breakdown catalog CRUD (add/edit/remove elements per scene) | BE-3 | Elements editable; version-locked after publish |
| Breakdown Service: `extractBreakdownFromVersion` gRPC (for publish saga Temporal activity) | BE-3 | Called by publish-saga worker; breakdown version created |
| Breakdown Service: NATS consumer — `script.checkpointed` → trigger re-detection on changed scenes | BE-3 | Changed scenes re-tagged within 30s of checkpoint event |
| Breakdown Service: GraphQL subgraph — BreakdownElement, BreakdownCategory types | BE-3 | `scene { breakdownElements { category name } }` works |
| AI Service: company consent model + DDL | ML | `ai.company_consents` table; `checkConsent()` working |
| AI Service: context assembler (Bible + timeline + surrounding scenes) | ML | `assembleContext()` returns cacheable prefix + fresh suffix |
| AI Service: model router (Haiku/Sonnet/Opus by feature tier) | ML | Dialogue suggestion → Sonnet; format check → Haiku |
| AI Service: Anthropic SDK integration with prompt caching | ML | Cache hits visible in response `usage.cache_read_input_tokens` |
| AI Service: AI ledger write (append-only, DB trigger immutability) | ML | Ledger row written; DELETE trigger fires if attempted |
| AI Service: dialogue suggestion feature (end-to-end) | ML | Writer requests suggestion → consent checked → context assembled → model called → guard → ledger written → suggestion displayed |
| AI Service: `format_correction` feature (always available, Tier 1) | ML | Format violations flagged without company consent check |
| AI Service: rate limiting (org quota + per-user soft limit via Redis Lua) | ML | Professional org quota of 500K tokens enforced; 429 returned when exhausted |
| AI Service: GraphQL subgraph — AISuggestion, AILedgerEntry types | ML | `requestAISuggestion` mutation returns suggestion + ledgerEntryId |
| Search Service: Elasticsearch index setup (BM25 + vector) | BE-3 | Index created; mapping for `script_content` + `embedding` fields |
| Search Service: `script.checkpointed` → re-index changed scenes | BE-3 | Scene text searchable within 60s of checkpoint |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| AI suggestion panel: request, display, accept/reject/modify | FE-1 | Writer clicks "Suggest dialogue" → panel shows 2-3 options → accept writes to editor |
| AI suggestion consent UI: per-feature opt-in toggles | FE-1 | Writer enables dialogue_suggestion; consent stored |
| Breakdown panel: per-scene element list, category filters | FE-2 | Breakdown auto-populated; elements editable |
| Breakdown: add/remove element manually | FE-2 | Manual element appears in breakdown; category selectable |
| Search bar (basic): full-text search across script content | FE-2 | Search returns relevant scenes; clicking navigates to scene in editor |

### Sprint 6 Definition of Done

- [ ] **Alpha milestone (Week 14):** 10 external writers invited; editor + bible + basic AI suggestions working
- [ ] Dialogue suggestion end-to-end: consent check → context assembly → Anthropic API → guard → ledger → UI
- [ ] Breakdown auto-detection runs on checkpoint; manual edits persist
- [ ] AI ledger entry written for every suggestion; immutability trigger prevents deletion
- [ ] Search returns results within 500ms for a 100-scene script
- [ ] Breakdown Service + AI Service + Search Service deployed to staging

---

## Sprint 7 — Scheduling Service + OR-Tools Solver (Weeks 15–16)

**Goal:** Scheduling service running. Stripboard generated from breakdown. Day-out-of-days calculated.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Atlas schema for `scheduling.*` tables | BE-3 | Tables: `shoot_days`, `scene_schedule_assignments`, `schedule_versions`, `dood_entries` |
| Scheduling Service: stripboard CRUD (create/read/update/delete shoot days) | BE-3 | Shoot day created; scenes assigned to day |
| Scheduling Service: OR-Tools constraint solver (scene ordering, company moves, location grouping) | BE-5 | Solver produces valid schedule for 30-scene script in < 5s |
| Scheduling Service: day-out-of-days (DOOD) calculation per character | BE-5 | DOOD table: character × shoot day matrix; first/last/hold/travel/work |
| Scheduling Service: manual override (1st AD drag-and-drop) always takes precedence over solver | BE-3 | Manual assignment not overwritten by solver re-run |
| Scheduling Service: `computeVersionDelta` + `rollbackDelta` gRPC (for publish saga) | BE-3 | Delta computed; rollback restores previous schedule state |
| Scheduling Service: GraphQL subgraph — ShootDay, SceneAssignment, DOOD types | BE-3 | `shootDays { date scenes { sceneNumber } }` works |
| Scheduling Service: NATS consumer — `breakdown.updated` → re-solve affected days | BE-3 | Solver re-runs on breakdown change; affected days marked `needs_review` |
| Proto definitions: SchedulingService complete | BE-3 | `packages/proto/scheduling.proto` complete |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Stripboard UI: shoot days as columns; scenes as draggable cards | FE-2 | Scenes drag between days; order preserved |
| Calendar view: shoot days on a calendar; click to open stripboard day | FE-2 | Calendar shows shoot days; click → day detail |
| DOOD table: character × shoot day matrix with color coding | FE-2 | W/H/T/F cells; exportable to CSV |
| Solve button: runs OR-Tools; proposes optimal schedule; user confirms | FE-2 | "Re-solve" button → solver runs → proposed schedule shown → confirm/reject |

### Sprint 7 Definition of Done

- [ ] OR-Tools solver produces valid schedule for test project (30 scenes, 5 locations, 8 characters) in < 5s
- [ ] Stripboard UI drag-and-drop works; manual overrides preserved across solver re-runs
- [ ] DOOD table calculated correctly for all characters
- [ ] Scheduling Service deployed to staging

---

## Sprint 8 — Budget Service + Publish Saga (Weeks 17–18)

**Goal:** Budget lines calculated from schedule. Full publish-to-production saga running end-to-end in Temporal.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Atlas schema for `budget.*` tables | BE-4 | Tables: `budget_lines`, `account_codes`, `budget_versions`, `budget_deltas` |
| Budget Service: budget line CRUD (above-the-line, below-the-line, post) | BE-4 | Budget lines created; account codes assigned |
| Budget Service: `computeVersionDelta` + `rollbackDelta` gRPC | BE-4 | Delta computed from schedule delta input |
| Budget Service: accounting export (EP Budgeting / Excel format) | BE-4 | Export produces XLSX with correct account code structure |
| Budget Service: GraphQL subgraph — BudgetLine, AccountCode, BudgetSummary types | BE-4 | `budget { totalAboveTheLine totalBelowTheLine }` works |
| Temporal: publish-saga worker deployment | BE-1 | Worker pod running; connects to Temporal Cloud |
| Publish saga: all 7 steps implemented with compensation stack | BE-1 | Workflow runs end-to-end (happy path) for test script |
| Publish saga: approval chain config + `workflow.approval_chain_configs` DDL | BE-1 | TV_EPISODE template loaded; approval steps fire notifications |
| Publish saga: approval signals + escalation timers | BE-1 | 24h escalation fires in time-skipping test; 48h auto-approve fires |
| Publish saga: `approval_requests` table; status updates from signal handler | BE-2 | Each approval step has a DB record; status updated on decision |
| Notification Service: email via SendGrid | BE-4 | Approval request email sent to approver |
| Notification Service: in-app WebSocket push | BE-4 | Browser receives notification within 2s of saga signal |
| Legal Service: stub — `validateScriptPolicy` always passes (full implementation Phase 5) | BE-4 | Stub returns `{ passed: true }` |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Budget panel: above/below-the-line table; edit budget lines | FE-2 | Budget lines editable; totals update |
| Publish button + confirmation modal | FE-1 | "Publish to Production" button → confirmation → saga started |
| Publish status tracker: Submitting → Validating → In Planning → Awaiting Approval | FE-1 | Status bar shows current saga step; updates via WebSocket push |
| Approval inbox: pending approvals for current user | FE-1 | User sees pending approval requests; Approve/Reject buttons |
| Approval detail: shows script version diff + schedule delta + budget delta | FE-1 | Approver can review changes before deciding |

### Sprint 8 Definition of Done

- [ ] Full publish saga runs end-to-end: lock → validate → breakdown → schedule → budget → approval → lock
- [ ] Approval email sent; in-app notification received
- [ ] Compensation fires correctly when approval is rejected (budget → schedule → breakdown → unlock)
- [ ] 48h auto-approve fires in time-skipping integration test
- [ ] Budget export produces valid XLSX

---

## Sprint 9 — Revision Workflow + Call Sheets (Weeks 19–20)

**Goal:** Color-coded revision workflow working. Call sheets generated and distributed. Revision distribution saga running.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Script Service: revision color state machine (13 colors per WGA spec) | BE-2 | `updateRevisionColor` mutation; locked pages cannot change color |
| Script Service: diff view between versions (unified diff at scene level) | BE-2 | `scriptVersionDiff(fromId, toId)` returns changed scene list with diffs |
| Revision distribution saga: generate colored pages PDF + per-recipient watermark + distribute | BE-1 | Saga completes; recipient receives watermarked PDF via email |
| Watermark Service: character spacing steganography in PDF | BE-5 | Invisible watermark embeds recipient ID; extraction recovers ID from PDF |
| Watermark Service: zero-width character steganography in FDX | BE-5 | FDX watermark encodes recipient ID in zero-width chars between dialogue |
| Watermark Service: `embedWatermarkPerRecipient` gRPC | BE-5 | Callable from revision-dist saga; returns watermarked S3 key |
| Call sheet saga: pull scenes + cast + crew + logistics → generate → AD approval → distribute | BE-1 | Call sheet PDF generated; sent for AD approval; distributed on approve |
| `packages/db` — Atlas schema for `call_sheet.*` tables | BE-4 | Tables: `call_sheets`, `call_sheet_items`, `crew_assignments` |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Revision color badge: script page shows current revision color | FE-1 | Color badge (Blue, Pink, Yellow…) visible in script header |
| Version history panel: list of locked versions; click → open diff view | FE-1 | Side-by-side diff shows added/changed/removed scenes |
| Call sheet viewer: rendered call sheet in app | FE-2 | Call sheet UI matches industry standard layout |
| Watermark indicator: export modal shows "Recipient watermark will be embedded" | FE-2 | Export generates watermarked PDF; warning shown if no consent |

### Sprint 9 Definition of Done

- [ ] Revision color changes tracked; locked versions immutable
- [ ] Watermarked PDF: embed → extract → recover recipient ID round-trip verified
- [ ] Call sheet generated and distributed to 3 test crew members
- [ ] Revision distribution saga deploys watermarked PDFs per recipient
- [ ] **Beta milestone target approaches (Week 24): 3 production teams using breakdown + scheduling**

---

## Sprint 10 — Supervisor Module + PowerSync Offline (Weeks 21–22)

**Goal:** Tauri on-set app functional. Script Supervisor can log takes offline. PowerSync syncs to cloud when connected.

### Backend / Platform

| Task | Owner | Done When |
|------|-------|-----------|
| Supervisor Service: Postgres schema (`supervisor.*` tables) | BE-3 | Tables created via Atlas; scenes, takes, deviations, assertions, photos |
| Supervisor Service: take CRUD + rating API | BE-3 | `logTake`, `rateTake` gRPC methods |
| Supervisor Service: continuity assertion CRUD | BE-3 | Assertions created; linked to story day |
| Supervisor Service: photo upload endpoint (presigned S3 URL) | BE-3 | `getPresignedUploadURL` → client uploads directly to S3 |
| Supervisor Service: GraphQL subgraph — Take, Deviation, ContinuityAssertion types | BE-3 | `scene { takes { takeNumber rating } }` works |
| Supervisor Service: NATS publisher — `supervisor.take.circled`, `supervisor.scene.completed` | BE-3 | Events published on sync; editorial consumers notified |
| PowerSync Sync Streams: role-based data routing (supervisor, AD, editorial) | DevOps | Each role receives only designated data on device |
| PowerSync cloud sync service: configured for `supervisor.*` schema | DevOps | Sync works from cloud Postgres → SQLite on test device |
| Conflict detection: circled take conflicts → manual review queue | BE-3 | Two concurrent circles → conflict entry created |

### Tauri App (On-Set Client)

| Task | Owner | Done When |
|------|-------|-----------|
| Tauri 2.x scaffold (`apps/onset`) — Rust backend + React frontend | FE-2 | `cargo tauri dev` runs; React UI loads in native window |
| SQLite setup + WAL mode + SQLCipher key derivation (Argon2id + Stronghold) | FE-2 | Encrypted DB opens; key derived from session token |
| PowerSync Rust SDK integration | FE-2 | `start_sync()` connects; SQLite tables populated from cloud |
| Tauri commands: `log_take`, `rate_take`, `create_continuity_assertion` | FE-2 | Commands callable from React via `invoke()` |
| Tauri command: `attach_continuity_photo` — capture + compress + enqueue upload | FE-2 | Photo compressed to JPEG 85%; thumbnail generated; upload queued |
| Sync status indicator: pending writes count + connectivity status | FE-2 | UI shows "12 changes pending sync" when offline |
| Scene list + take logging UI | FE-2 | Script Supervisor can select scene, log take, rate (circle/print/no_print) |
| Offline simulation test: disconnect → log 50 takes → reconnect → all synced | FE-2 | All 50 takes appear in cloud Postgres after reconnect |

### Sprint 10 Definition of Done

- [ ] Tauri app runs on macOS; script supervisor can log takes offline for 1 hour; all sync to cloud on reconnect
- [ ] PowerSync sync works for supervisor + AD roles (different data subsets)
- [ ] Photo upload queued offline; uploads on reconnect; S3 key written to DB
- [ ] SQLCipher encryption verified: DB file is unreadable without key
- [ ] Supervisor Service deployed to staging; all gRPC methods passing integration tests

---

## Sprint 11 — SeriesTimeline + Continuity Service (Weeks 23–24)

**Goal:** Cross-episode continuity graph operational. Continuity assertions from on-set sync to SeriesTimeline. Violations detected and surfaced.

**Beta milestone (Week 24): 3 production teams using breakdown + scheduling + on-set module.**

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Atlas schema for `series_timeline.*` tables | BE-5 | Tables: `timeline_story_days`, `story_day_scenes`, `continuity_assertions`, `assertion_violations` |
| Continuity Service: story day CRUD | BE-5 | Story day created; scenes linked; time-of-day slot assigned |
| Continuity Service: continuity assertion ingestion from supervisor sync | BE-5 | `supervisor.assertion.synced` NATS event → assertion stored in SeriesTimeline |
| Continuity Service: violation detection (wardrobe, injuries, props, relationships) | BE-5 | Contradictory assertions on same subject + story day → violation created |
| Continuity Service: cross-episode violation surfacing | BE-5 | Violation in episode 3 shows impact on episode 5 |
| Continuity Service: AI-assisted violation check (calls AI Service for ambiguous cases) | ML | AI checks violations where rule-based detection is inconclusive |
| Continuity Service: GraphQL subgraph — StoryDay, ContinuityAssertion, Violation types | BE-5 | `storyDay { assertions { subject text } violations { description } }` works |
| Continuity Service: NATS consumer — `supervisor.assertion.synced` | BE-5 | Consumer subscribed; assertions processed within 10s |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| SeriesTimeline view: story days on a timeline; click → assertions for that day | FE-1 | Timeline rendered; story days clickable |
| Continuity violations panel: list of violations with severity | FE-1 | Violations show subject, description, affected episodes |
| Continuity assertion detail: photo thumbnails + linked takes | FE-2 | Continuity photo visible next to assertion text |
| Violation resolution workflow: mark as resolved / intentional / error | FE-2 | Violation status updatable; reason recorded |

### Sprint 11 Definition of Done

- [ ] On-set continuity assertion → syncs to cloud → appears in SeriesTimeline → violation detected if contradictory
- [ ] Cross-episode violation: character injury in episode 3 flagged as inconsistent with healthy appearance in episode 4 (story day 6)
- [ ] Beta milestone: 3 pilot production teams onboarded with breakdown + scheduling + on-set module access
- [ ] Continuity Service deployed to staging

---

## Sprint 12 — Character Voice Registry + AI Deep Features (Weeks 25–26)

**Goal:** Character voice registry complete with few-shot dialogue suggestions. Semantic cache operational. AI coverage analysis working.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Bible Service: voice profile reference scene management (3–5 scenes per character) | BE-2 | Reference scenes selectable; used in AI context assembly |
| AI Service: semantic cache (pgvector ANN search; 0.97 threshold) | ML | Second identical dialogue request returns cache hit; cost_usd = 0 |
| AI Service: `consistency_check` feature (non-generative; Tier 1) | ML | Consistency check returns list of canon violations in current scene |
| AI Service: `coverage_analysis` feature (full script analysis; Tier 3) | ML | Coverage report: logline, synopsis, structure, character assessment, recommendation |
| AI Service: `structure_analysis` feature (act breaks, pacing; Tier 3) | ML | Structure report with act break positions and pacing assessment |
| AI Service: semantic cache TTL (24h creative, 7d analytical) | ML | Cache entries expire correctly; expired entries not returned |
| AI Service: C2PA manifest embedding in PDF export | ML | Exported PDF contains C2PA manifest with `com.scriptos.ai_involvement` assertion |
| AI Service: WGA compliance report export (per-project, period, JSON/PDF/CSV) | ML | `generateWGAReport()` produces correct JSON for test project |
| Search Service: hybrid BM25 + vector search with RRF | BE-3 | Hybrid search returns more relevant results than BM25 alone (human eval) |
| Search Service: Elasticsearch vector mapping + embedding pipeline on checkpoint | BE-3 | Scenes have embeddings in ES; ANN search works |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Voice profile editor: vocabulary level, patterns, forbidden words | FE-1 | Editable; violation warnings shown when AI suggestion contains forbidden word |
| Reference scene selector: pick 3-5 exemplar scenes per character | FE-1 | Scenes selectable from script; stored as voice profile |
| AI suggestion panel: voice violation warnings inline | FE-1 | Warning badge on suggestion: "WALTER would not say 'literally'" |
| Coverage analysis UI: full coverage report rendered | FE-2 | Report rendered with logline, structure diagram, recommendation badge |
| AI usage dashboard (org admin): token consumption, cost, acceptance rate | FE-2 | Admin sees org-wide AI usage; per-user breakdown |

### Sprint 12 Definition of Done

- [ ] Second dialogue suggestion for same query hits semantic cache (verified by ledger entry `model_id = 'semantic_cache'`)
- [ ] Voice violation warning shown when AI suggests forbidden word
- [ ] Coverage analysis produces structured report for a 90-page test script
- [ ] Hybrid search (BM25 + vector RRF) returns correct results for semantic queries ("scene where Walter threatens Jesse")
- [ ] WGA compliance report exports correctly for a project with 50+ AI interactions

---

## Sprint 13 — Tauri On-Set Polish + Basecamp Mode (Weeks 27–28)

**Goal:** On-set Tauri app production-ready. Basecamp self-hosted PowerSync mode tested. Script Supervisor day-end turnover generation.

### Backend / DevOps

| Task | Owner | Done When |
|------|-------|-----------|
| Supervisor Service: day-end turnover package generation (PDF with lined script, takes, deviations) | BE-3 | Turnover PDF generated on `supervisor.scene.completed` event |
| Supervisor Service: faced pages export (supervisor marked script pages) | BE-3 | Faced pages PDF exportable |
| Supervisor Service: conflict resolution UI data — manual review queue API | BE-3 | `/supervisor/conflicts` endpoint returns pending conflicts |
| Basecamp Docker Compose (`docker-compose.basecamp.yml`) | DevOps | Self-hosted PowerSync + local Postgres runs on a laptop |
| Basecamp setup script (`scripts/basecamp-setup.sh`) | DevOps | Script dumps production data for a project; brings up services |
| Basecamp sync test: offline for 72h simulation → reconnect → verify no data loss | DevOps | 72-hour offline test passes (automated in CI) |
| Loro CRDT: Tauri SQLite persistence for continuity notes | FE-2 | Notes persist across Tauri app restarts; CRDT state in `crdt_doc_states` |
| On-set CRDT sync: WebSocket → Loro → SQLite → WebSocket (when connected) | FE-2 | Two on-set devices see mutual continuity note edits when on local WiFi |

### Tauri App Polish

| Task | Owner | Done When |
|------|-------|-----------|
| Offline indicator: prominent banner "OFFLINE — 23 changes pending" | FE-2 | Banner shown when PowerSync disconnected |
| Photo gallery: grid view of continuity photos for a scene | FE-2 | Photos browsable; caption editable |
| Conflict resolution UI: two circled takes flagged; supervisor resolves | FE-2 | Supervisor picks the "true" circle; resolution syncs to cloud |
| Turnover package: generate and share from Tauri | FE-2 | Day-end turnover PDF generated and saved to Desktop |
| App auto-update: Tauri updater checks for new version on launch | DevOps | New version downloaded and applied automatically |

### Sprint 13 Definition of Done

- [ ] 72-hour offline test: 300 takes, 100 assertions, 50 photos — all sync correctly on reconnect
- [ ] Basecamp mode: 3 Tauri clients sync via local WiFi (no internet) to basecamp laptop
- [ ] Day-end turnover PDF generated for a test shooting day
- [ ] Tauri app auto-updates to new version

---

## Sprint 14 — Editorial Round-Trip (Weeks 29–30)

**Goal:** OTIO + AAF export working. Correlation DB populated. Editorial re-import diff working. Change review workflow running.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| `packages/db` — Atlas schema for `editorial.*` tables | BE-4 | Tables: `export_batches`, `correlation_records`, `reimport_batches` |
| Editorial: OTIO export from Script AST (with `scriptos.*` custom metadata) | BE-4 | OTIO file generated; clips have `scriptos.scene_id` in metadata |
| Editorial: AAF export for Avid delivery | BE-4 | AAF file generated; `SCRIPTOS_SCENE_ID` custom data in SourceMob |
| Editorial: EDL export (CMX 3600 format) | BE-4 | EDL file generated; reel names ≤ 8 chars |
| Editorial: correlation DB populated on export | BE-4 | One record per scene in `editorial.correlation_records` |
| Editorial: format normalizer (AAF/EDL → OTIO via Python subprocess) | BE-4 | AAF input → normalized OTIO output; EDL input → OTIO |
| Editorial: diff engine (previous OTIO vs re-imported OTIO) | BE-4 | Omissions, reorders, take changes, untracked clips detected correctly |
| Editorial re-import saga (Temporal): normalize → diff → review → apply | BE-1 | Saga completes; approved operations applied to script graph |
| Supervisor Service: `setCircledTake` gRPC (for take selection change apply) | BE-3 | Circled take updated from editorial re-import |
| Frame.io OAuth integration: connect Frame.io account | BE-4 | OAuth flow works; `frameio_credentials` row stored |
| Frame.io webhook handler: comment.created → ReviewComment in DB | BE-4 | Frame.io comment appears in ScriptOS review comments within 5s |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Editorial export modal: choose formats (OTIO/AAF/EDL); download | FE-2 | Export modal; formats selectable; download triggers |
| Re-import upload: drag-and-drop OTIO/AAF file | FE-2 | File uploaded; saga started; progress shown |
| Change review UI: list of detected changes; approve/reject per operation | FE-1 | Operations listed; approve/reject toggles; submit decision |
| Untracked clip queue: flag unrecognized clips for manual mapping | FE-1 | Untracked clips listed; supervisor maps to scene manually |
| Frame.io connect: OAuth button in integration settings | FE-2 | "Connect Frame.io" → OAuth → credentials stored → integration active |
| Review comments panel: comments from Frame.io linked to takes | FE-2 | Comments show timecode, reviewer name, take reference |

### Sprint 14 Definition of Done

- [ ] OTIO export → simulated NLE omit + reorder → re-import → diff detects correct operations
- [ ] Correlation DB fallback matching: after stripping OTIO metadata, correlation DB still resolves scene IDs
- [ ] Editorial re-import saga: approved operations applied to script graph (scene status = 'omitted')
- [ ] Frame.io webhook comment received and linked to take within 5s

---

## Sprint 15 — Legal / Rights Service + Compliance (Weeks 31–32)

**Goal:** Legal service fully implemented. Rights tags, NDA gates, legal holds. SOC 2 evidence collection started. GDPR data export.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Legal Service: rights tags on script versions (option, purchase, underlying rights) | BE-4 | Rights tags CRUD; linked to script versions |
| Legal Service: NDA gate — block access to script without signed NDA | BE-4 | Script access denied if viewer has not signed project NDA |
| Legal Service: legal hold — prevent deletion of assets under hold | BE-4 | Held assets return 403 on delete; hold reason logged |
| Legal Service: `validateScriptPolicy` gRPC (full implementation — replaces Sprint 8 stub) | BE-4 | Policy violations returned for rights gaps, NDA missing |
| Legal Service: GraphQL subgraph — RightsTag, NDARecord, LegalHold types | BE-4 | `script { rightsStatus { type status } }` works |
| Auth Service: GDPR data export (all data for a user, JSON format) | BE-3 | `exportUserData(userId)` returns complete data package |
| Auth Service: account deletion (right to be forgotten — cascade to all services) | BE-3 | User deletion cascades; AI ledger entries anonymized (not deleted — WGA compliance) |
| Watermark Service: forensic watermark extraction + reporter | BE-5 | Given a leaked PDF, `extractWatermark(pdf)` returns recipient ID |
| Watermark Service: watermark extraction API endpoint (for security team) | BE-5 | API callable; returns recipient ID from uploaded PDF |
| SOC 2 evidence: audit log completeness check | DevOps | All mutations have audit log entries; coverage > 99% |
| Penetration test: schedule external pentest for Sprint 16 | DevOps | Pentest vendor engaged; scope defined |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Rights status panel in script settings | FE-2 | Rights tags viewable; NDA status shown |
| NDA gate modal: "Sign NDA to access this script" | FE-1 | Unsigned writers blocked; signing flow (DocuSign integration stub) |
| Legal hold indicator: locked scripts show "Legal Hold — contact legal@company" | FE-1 | Held scripts show indicator; export disabled |
| GDPR: account settings → "Download my data" | FE-2 | JSON export download triggers |

### Sprint 15 Definition of Done

- [ ] Legal policy validation blocks publish when rights are missing or NDA not signed
- [ ] Watermark extraction recovers recipient ID from a watermarked test PDF
- [ ] GDPR data export produces complete JSON package for a test user
- [ ] Legal holds block deletion; hold reason in audit log

---

## Sprint 16 — Security Hardening + Accessibility (Weeks 33–34)

**Goal:** Penetration test findings resolved. WCAG 2.2 AA audit passed. SOC 2 readiness report completed.

### Backend / DevOps

| Task | Owner | Done When |
|------|-------|-----------|
| Penetration test remediation (findings from external pentest) | All | All critical/high findings resolved; medium findings triaged |
| Input validation audit: all GraphQL mutations validated with Zod | BE-1 | No mutation accepts unvalidated user input |
| Rate limiting: global API rate limiting at HTTP Gateway | DevOps | 1000 req/min per user; 429 returned with Retry-After header |
| RBAC audit: verify no role escalation paths | BE-3 | Automated RBAC matrix test covers all role × resource combinations |
| Database encryption at rest: Cloud SQL encrypted by default; verify key management | DevOps | Encryption key in Cloud KMS; rotation schedule set |
| Secrets rotation: rotate all API keys + DB passwords | DevOps | All secrets rotated; old secrets revoked |
| SOC 2 readiness: evidence collection complete | DevOps | SOC 2 Type I report package assembled |
| Backup verification: Postgres + Neo4j + S3 backups restore successfully | DevOps | Point-in-time restore tested for each data store |
| Load test: 500 concurrent users; 50 concurrent CRDT editors | DevOps | P99 latency < 500ms; no errors under load |

### Frontend / Accessibility

| Task | Owner | Done When |
|------|-------|-----------|
| WCAG 2.2 AA audit (external auditor) | FE-1 | Audit report received; findings prioritized |
| WCAG 2.2 AA remediation: keyboard navigation, screen reader support, contrast | FE-1 | All Level A + AA criteria pass axe-core automated scan |
| WCAG: editor accessible to screen readers (aria-live for collaboration events) | FE-1 | NVDA/VoiceOver can navigate script structure |
| WCAG: Tauri on-set app keyboard navigation | FE-2 | Take logging operable without mouse |
| Performance audit: Lighthouse score ≥ 90 on editor page | FE-1 | Lighthouse passes |

### Sprint 16 Definition of Done

- [ ] All pentest critical/high findings resolved
- [ ] WCAG 2.2 AA audit passed (all Level AA criteria)
- [ ] Load test: 500 concurrent users; P99 < 500ms; error rate < 0.1%
- [ ] SOC 2 Type I readiness package complete

---

## Sprint 17 — Story Development + P1 Features (Weeks 35–36)

**Goal:** Story development UI (beat boards, index cards). Accounting export. Hybrid search hardened. Cross-episode continuity UI polished.

### Backend

| Task | Owner | Done When |
|------|-------|-----------|
| Story Development: beat board data model (beats → scenes, beat type, arc assignment) | BE-5 | Beat nodes in Neo4j; linked to scene in script |
| Story Development: arc CRUD (protagonist arc, antagonist arc, subplot) | BE-5 | Arcs created; beats assigned to arc |
| Budget Service: accounting export (EP Budgeting format XLSX) | BE-4 | Exported XLSX validates against EP Budgeting template |
| Hybrid search: Elasticsearch tuning (BM25 b/k1 parameters, vector weight in RRF) | BE-3 | Recall@10 improved ≥ 15% on eval set after tuning |
| Hybrid search: faceted search (filter by episode, scene type, character) | BE-3 | Facets work; character filter returns only scenes with that character |
| Continuity Service: cross-episode timeline view data API | BE-5 | `timelineView(projectId)` returns story days across all episodes |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Beat board UI: index cards on a virtual corkboard; drag to reorder | FE-1 | Cards draggable; scene link shows linked script scene |
| Arc view: character arcs visualized as swimlane across story days | FE-1 | Arc swimlane rendered; beat cards placed at story day positions |
| Accounting export: budget page → "Export to EP Budgeting" | FE-2 | Export triggers; XLSX downloaded |
| Advanced search: filters by character, location, episode, scene status | FE-2 | Filtered search returns correct results |
| Cross-episode continuity timeline: full-series view | FE-1 | All story days across episodes on one timeline |

### Sprint 17 Definition of Done

- [ ] Beat board UI: cards reorderable; linked to script scenes
- [ ] Cross-episode continuity timeline rendered for a 3-episode test project
- [ ] Accounting export produces valid EP Budgeting XLSX
- [ ] Hybrid search tuned; recall improved; faceted filters working

---

## Sprint 18 — GA Hardening + Launch Prep (Weeks 37–38)

**Goal:** General Availability. All P0 and P1 features complete. Monitoring, alerts, and runbooks in place. Onboarding flows polished.

### Backend / DevOps

| Task | Owner | Done When |
|------|-------|-----------|
| Production environment: separate GKE cluster from staging | DevOps | Production cluster provisioned; DNS configured |
| Multi-region DR: Postgres read replica in secondary region | DevOps | Failover tested; RPO < 5 min; RTO < 15 min |
| Alerting: PagerDuty integration; alerts for P99 > 1s, error rate > 1%, disk > 80% | DevOps | On-call rotation configured; test alert fires and pages |
| Runbooks: written for all Tier 1 incidents (DB failover, NATS outage, Temporal stuck workflow) | DevOps | Runbooks in Confluence/Notion; reviewed by team |
| Atlas migration process: documented; rollback procedure tested | BE-1 | Migration apply + rollback round-trip tested on staging |
| Unleash feature flags: GA flags for all P2 features (not yet built) | DevOps | P2 features behind flags; default off |
| y-redis migration: Hocuspocus → y-redis for CRDT horizontal scaling | BE-1 | y-redis Deployment + HPA + Worker running; CRDT sync works through y-redis |
| Pricing enforcement: Indie tier blocks AI generation; Professional token quota enforced | BE-1 | Test accounts in each tier behave correctly |
| SCIM provisioning: full implementation for enterprise SSO orgs | BE-3 | SCIM endpoint creates/deactivates users from Okta |

### Frontend

| Task | Owner | Done When |
|------|-------|-----------|
| Onboarding flow: new user → create org → create project → import script → invite collaborator | FE-1 | New user can complete onboarding in < 5 minutes |
| Empty state UX for all panels (no scenes, no breakdown, no schedule) | FE-2 | Helpful empty states with CTA buttons |
| In-app help: tooltips, keyboard shortcut cheat sheet | FE-1 | `?` key opens shortcut reference |
| Mobile-responsive web (tablet breakpoint) | FE-2 | Core pages usable on iPad in portrait |
| Error states: graceful degradation when services are unavailable | FE-1 | API error → toast notification; no blank screens |

### Sprint 18 Definition of Done

- [ ] Production environment live; DNS pointing to production cluster
- [ ] DR failover tested; RPO and RTO targets met
- [ ] Onboarding flow: new user completes in < 5 minutes in user test
- [ ] y-redis deployed; 20 concurrent CRDT editors tested under y-redis
- [ ] All Tier 1 runbooks written and reviewed
- [ ] **GA milestone: Full platform available**

---

## Key Milestones

| Milestone | Sprint | Target Week | Success Criteria |
|-----------|--------|-------------|-----------------|
| **Dev dogfood** | Sprint 3 end | Week 8 | Engineering team writes scripts in ScriptOS daily |
| **Alpha (writers)** | Sprint 6 end | Week 14 | 10 external writers using editor + bible + AI |
| **Beta (production teams)** | Sprint 11 end | Week 24 | 3 production teams using breakdown + scheduling + on-set |
| **GA** | Sprint 18 end | Week 38 | Full platform; production environment live |

---

## Service Deployment Sequence Summary

| Sprint | Services First Deployed to Staging |
|--------|-----------------------------------|
| 0 | Infrastructure only (no services) |
| 1 | Auth Service, HTTP Gateway, WebSocket Gateway stub |
| 2 | Script Service |
| 3 | Collaboration Service |
| 4 | Import Service |
| 5 | Bible Service |
| 6 | Breakdown Service, AI Service, Search Service |
| 7 | Scheduling Service |
| 8 | Budget Service, Notification Service, Legal Service (stub), Temporal Workers (publish-saga, import-saga) |
| 9 | Watermark Service, Temporal Workers (revision-dist, call-sheet) |
| 10 | Supervisor Service, Tauri on-set app |
| 11 | Continuity Service |
| 12 | (AI + Search deep features — no new services) |
| 13 | (Tauri polish — no new services) |
| 14 | Editorial service (supervisor), Frame.io integration |
| 15 | Legal Service (full), GDPR endpoints |
| 16–18 | (Hardening, no new services) |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation | Sprint Trigger |
|------|-----------|--------|-----------|----------------|
| Loro POC fails (CRDT gate) | Medium | Critical | Full Yjs fallback spec in wiki/04; 3-week contingency buffer built into Sprint 3 | Sprint 3 week 1 |
| Temporal Cloud pricing exceeds $2K/month | Low | Medium | Self-hosted Temporal as fallback; evaluated at Sprint 8 | Sprint 8 |
| OR-Tools solver too slow for large productions (200+ scenes) | Low | Medium | Incremental solve (only re-solve affected days); parallelise via OR-Tools multi-thread | Sprint 7 |
| PowerSync licensing changes / service instability | Low | High | Apache 2.0 self-hosted fallback always available; basecamp mode already uses it | Sprint 10 |
| OpenAI embedding API rate limits during bulk indexing | Medium | Low | Exponential backoff; batch size tuning; fallback to local BGE-M3 for enterprise | Sprint 6 |
| Neo4j Aura data egress costs exceed budget | Low | Medium | Switch to self-hosted Neo4j on GKE if Aura costs > $800/month | Sprint 5 |
| y-redis AGPL commercial license cost | Low | Low | Budget provisioned; Hocuspocus (MIT) covers startup phase until Sprint 18 | Sprint 18 |
| Frame.io API changes break webhook integration | Medium | Low | Webhook contract pinned to API version; monitor Frame.io changelog | Sprint 14 |
| WCAG audit reveals major structural issues | Medium | Medium | Accessibility review in Sprint 6 (earlier check); remediation budget in Sprint 16 | Sprint 16 |
| External pentest reveals critical vulnerability | Low | Critical | Remediation sprint carved out between 16 and GA; delay GA if needed | Sprint 16 |
| Tauri iOS not production-ready by Q3 2026 | Medium | Low | React Native fallback ready; UI code shared; decision gate in Q3 2026 | Post-GA |

---

## Inter-Sprint Dependency Flags

Dependencies that block work if not met:

| Blocked Sprint | Blocked Task | Blocking Sprint | Blocking Deliverable |
|----------------|-------------|----------------|---------------------|
| Sprint 3 | CRDT sync | Sprint 2 | Script Service `LockDraft` + `CheckpointCRDT` gRPC |
| Sprint 5 | Bible ↔ Script linking | Sprint 2 | Script Service `script.checkpointed` NATS event |
| Sprint 6 | AI context assembly | Sprint 5 | Bible Service `getCharacterFacts`, `getVoiceProfiles` gRPC |
| Sprint 7 | Schedule computation | Sprint 6 | Breakdown Service `extractBreakdown` gRPC |
| Sprint 8 | Budget delta | Sprint 7 | Scheduling Service `computeVersionDelta` gRPC |
| Sprint 8 | Publish saga | Sprint 6, 7 | All activities must be callable; Legal stub must exist |
| Sprint 10 | PowerSync sync | Sprint 8 | Supervisor Service must be running; `supervisor.*` schema applied |
| Sprint 11 | Continuity violation | Sprint 10 | Supervisor Service must publish `supervisor.assertion.synced` NATS |
| Sprint 14 | Editorial re-import | Sprint 9 | OTIO export must exist; correlation DB must be populated |
| Sprint 15 | Full legal validation | Sprint 8 | Legal stub must be replaced before GA |
| Sprint 18 | y-redis migration | Sprint 3 | Hocuspocus must be working; y-redis swaps in on the same WS interface |

---

## Decisions

**Team size assumption — 8–12 engineers: 5–7 backend, 2–3 frontend, 1 DevOps/Platform, 1 ML/AI.**
Phase timelines assume 8 engineers. A smaller team extends each sprint by 50–100%. A larger team compresses Phases 2–4 but with diminishing returns past ~15 engineers given coordination overhead at this stage.

**Sprint length — 2 weeks.**
Bi-weekly sprints balance planning overhead against delivery cadence. 1-week sprints are too granular for infrastructure-heavy tasks (database migrations, Kubernetes setup). 3-week sprints are too long to detect integration problems early. Bi-weekly gives two natural checkpoint reviews per month.

**Milestones committed externally only from Alpha (Week 14) onward.**
No external commitment before Alpha. The data model, CRDT POC, and system design must be validated first. After Alpha (10 writers), Beta (3 production teams) is committable to pilot partners. GA commitment made after Beta learnings confirm the roadmap is realistic.

**y-redis migration deferred to Sprint 18 (GA hardening).**
Hocuspocus (MIT) is sufficient for the Alpha and Beta phases. y-redis is needed for horizontal WebSocket scaling — a production concern. Migrating before GA ensures the production environment is running the scale-ready configuration, not Hocuspocus. The WS interface is identical; the migration is a Kubernetes deployment swap, not an application change.

**SCIM provisioning deferred to Sprint 18.**
Enterprise SSO orgs need SCIM for automated user provisioning/deprovisioning. SCIM is not needed for Alpha or Beta (hand-managed invites). Sprint 18 GA hardening includes full SCIM for enterprise onboarding at GA.

**Legal Service stub in Sprint 8, full implementation in Sprint 15.**
The publish saga needs a legal validation activity to run end-to-end. A stub that always passes is sufficient for Sprint 8 Beta testing with pilot teams. Full rights tag + NDA gate implementation is a Sprint 15 deliverable — before GA but after Beta user feedback has validated the publish workflow.

**P2 features (table read TTS, multilingual, marketplace) — not in this roadmap.**
P2 features are behind Unleash feature flags from Sprint 18. They are not planned to sprint level here — they require separate roadmap planning after GA user feedback determines priority and sequencing.
