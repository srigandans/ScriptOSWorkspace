# 17 — P2 Feature Roadmap

> **Status:** Planned post-GA. P2 features are behind Unleash feature flags from Sprint 18. No sprint-level plans exist until GA user feedback determines priority order. This document captures intent, dependencies, scope, and sequencing logic for each P2 feature so the next roadmap cycle can begin from a solid foundation.

---

## What P2 Means

P2 features are **adjacent monetizable workflows** — not core platform differentiation, but significant value-adds that expand the platform's reach into adjacent parts of the production pipeline. They depend on the P0/P1 platform being stable and adopted.

None of these ship before GA. All are behind Unleash feature flags at GA. Priority order after GA is determined by:
1. Pilot user demand (which do Beta users ask for most?)
2. Revenue impact (which unlocks the next pricing tier or upsell?)
3. Technical readiness (which has the lowest build cost given what P0/P1 already delivers?)

---

## P2 Feature List

| Feature | One-Line Description | Primary Persona | Revenue Model |
|---------|---------------------|----------------|---------------|
| **Table Read TTS** | AI-voiced read-through of the full script per character | Writers, Directors | Studio tier add-on |
| **Location Scouting** | Location discovery, comparison, and permitting workflow | Location Manager, Producer | Professional + Studio |
| **Multilingual Workflows** | Script translation, dual-language scripts, localized exports | International productions, distributors | Studio tier; per-language |
| **Marketing & Pitch Generation** | One-pagers, loglines, lookbooks, pitch deck copy from the script | Writers, Producers, Agents | Professional + Studio |
| **Marketplace** | Third-party plugin and template ecosystem | Developers, Production companies | Revenue share |
| **Native Dailies Review Player** | In-app video player with frame-linked annotations and AI take selection | Editors, Directors, Producers | Studio tier |
| **Evercast Connector** | Remote collaboration viewing sessions linked to script takes | Directors, Producers (remote) | Studio tier add-on |

---

## 1. Table Read TTS

### What It Is

ScriptOS reads the script aloud, assigning a distinct AI-generated voice to each character. The output is an audio file (and optionally a video with scrolling script) that writers and directors can listen to without assembling the full cast.

Industry context: A table read is typically a half-day or full-day event with the entire cast present. TTS eliminates the scheduling cost for early-draft reads, iterations, and remote productions.

### User Flow

```
Writer opens a script → clicks "Generate Table Read"
       │
       ▼
TTS service assigns a voice model to each character
(drawn from Character Voice Registry + voice selection UI)
       │
       ▼
Script is segmented by character line
       │
       ▼
TTS synthesis per character (ElevenLabs / OpenAI TTS / Azure Neural)
       │
       ▼
Audio segments stitched: Action lines (narrator voice) + Dialogue (character voices)
       │
       ▼
Output: MP3 + optional SRT subtitle track
       │
       ▼
Shared with director / producers via review link (Frame.io or native)
```

### Dependencies

| Dependency | From P0/P1 | Why |
|------------|-----------|-----|
| Character Voice Registry | Sprint 12 (P1) | Voice profiles define vocabulary + tone; used as prompts for voice selection |
| Script AST | Sprint 2 (P0) | Segmentation by character and action requires structured AST |
| Frame.io connector | Sprint 14 (P1) | Audio review distribution |
| AI Service | Sprint 6 (P0) | Character voice profiles feed TTS voice selection |

### Technical Design

```typescript
// services/tts/ (new P2 service)

export interface TableReadRequest {
  scriptVersionId: string;
  projectId: string;
  voiceAssignments: VoiceAssignment[];   // character_id → voice_id
  options: {
    narratorVoice: string;               // for action lines
    paceMultiplier: number;              // 0.8 = slower, 1.2 = faster
    includeParentheticals: boolean;
    outputFormat: 'mp3' | 'wav';
  };
}

export interface VoiceAssignment {
  characterId: string;
  characterName: string;
  voiceProvider: 'elevenlabs' | 'openai' | 'azure';
  voiceId: string;                       // provider-specific voice ID
  stability?: number;                    // ElevenLabs: 0–1
  similarityBoost?: number;             // ElevenLabs: 0–1
}

// Synthesis pipeline:
// 1. Load script AST → flatten to ordered segments [{type: 'action'|'dialogue', text, characterId}]
// 2. For each segment: call TTS provider API with voice assignment
// 3. Store audio chunks in S3 as they arrive (streaming synthesis)
// 4. Stitch chunks with ffmpeg (1s pause between scenes, 0.3s between lines)
// 5. Generate SRT subtitle file from segment timestamps
// 6. Upload final MP3 + SRT to S3; notify via NATS: tts.table_read.completed
```

### TTS Provider Strategy

| Provider | Strength | Weakness | Use Case |
|---------|---------|---------|---------|
| ElevenLabs | Most expressive; voice cloning | Cost ($0.30/1K chars) | Studio tier; premium voices |
| OpenAI TTS | Cheap ($0.015/1K chars); reliable | Less expressive | Professional tier; quick reads |
| Azure Neural | Enterprise data residency | Setup complexity | Enterprise single-tenant |

**Routing:** OpenAI TTS for Professional tier. ElevenLabs for Studio tier or when writer requests "premium read." Azure Neural for enterprise data-residency clients.

### Scope Boundaries

**In scope:**
- Script → MP3 table read per character voice
- Voice selection UI (match voice profiles from registry)
- Audio review via Frame.io share link
- SRT subtitle export

**Out of scope (v1 of TTS):**
- Real-time TTS as writer types (too expensive, too distracting)
- Lip-sync video generation
- Actor voice cloning (legal complexity — deferred, requires explicit actor consent framework)
- Background music / ambient sound

### Pricing

Studio tier add-on: $0.005 per script page per table read generation (a 90-page script ≈ $0.45 per read). Included free: 5 reads/month per project in Studio tier. Professional tier: not included; available as metered add-on at same rate.

---

## 2. Location Scouting

### What It Is

A structured workflow for discovering, evaluating, shortlisting, and managing film locations — integrated with the breakdown (which locations are needed) and schedule (which shoot days need which locations).

### User Flow

```
Location Manager opens Location Scouting module
       │
       ▼
Breakdown-derived location requirements shown
(all distinct locations from AST, with scene count + shoot day requirements)
       │
       ▼
Location Manager creates Location records:
- Name, address, GPS coordinates
- Photos (uploaded or from Google Street View)
- Permitting status, contact, availability windows
- Estimated cost (day rate, permit fee)
- Notes (power access, parking, noise issues)
       │
       ▼
Locations compared against schedule requirements
(does this location work for the shoot days assigned to these scenes?)
       │
       ▼
Location shortlisted → Director review → Confirmed
       │
       ▼
Confirmed location linked to scenes in breakdown + schedule
       │
       ▼
Call sheet generator pulls confirmed location address/logistics
```

### Dependencies

| Dependency | From | Why |
|-----------|------|-----|
| Breakdown Service | Sprint 6 (P0) | Location requirements derived from breakdown element catalog |
| Scheduling Service | Sprint 7 (P0) | Location availability must align with scheduled shoot days |
| Search Service | Sprint 6 (P0) | Location discovery uses search (query: "INT. LAB" → suggest real lab locations) |
| Call Sheet saga | Sprint 9 (P0) | Confirmed location address feeds call sheet generation |

### Data Model

```typescript
interface Location {
  id: string;
  project_id: string;
  name: string;                          // "Abandoned Warehouse, Burbank"
  address: string;
  coordinates: { lat: number; lng: number };
  location_type: 'practical' | 'stage' | 'vfx_replacement' | 'backlot';
  status: 'prospecting' | 'shortlisted' | 'confirmed' | 'rejected';
  day_rate_usd: number | null;
  permit_fee_usd: number | null;
  permit_status: 'not_required' | 'pending' | 'approved' | 'denied';
  availability_windows: DateRange[];    // dates the location is available
  contact_name: string | null;
  contact_email: string | null;
  notes: string;
  photo_s3_keys: string[];
  linked_scene_ids: string[];           // which scenes are planned here
  confirmed_shoot_days: string[];       // which shoot days this location covers
}
```

### External Integrations

- **Google Maps API**: address geocoding, Street View thumbnails, travel time between locations (for company move planning)
- **Google Places API**: search for location types near a given area
- **Permit platforms**: stub for future integration with FilmLA, NYC Mayor's Office of Media; manual workflow at launch

### Scope Boundaries

**In scope:** Location record management, photo uploads, schedule compatibility check, call sheet integration.

**Out of scope (v1):** Automated permit filing, real-time location availability from third-party platforms, VR location scouting (360° tours).

---

## 3. Multilingual Workflows

### What It Is

Full support for writing, distributing, and managing scripts in multiple languages — including side-by-side dual-language scripts, AI-assisted translation, localized PDF export, and distributor-specific subtitle/dubbing packages.

### Why It's Complex

This is not just "translate the dialogue." Industry multilingual workflows include:
- **Dubbing scripts**: rewritten for lip-sync (mouth movement constraints), not literal translation
- **Subtitle scripts**: timed, character-limited per line, distinct from dialogue on the page
- **Continuity across languages**: character names and lore references must be consistent
- **Credits localization**: title pages differ per territory
- **Format compliance**: some territories have different screenplay format standards

### User Flow

```
Producer adds a language to a project → selects translation workflow
       │
       ├─► AI-assisted translation (Tier 2 — Sonnet)
       │     → writer reviews + edits each scene
       │     → accepted scenes committed to localized script version
       │
       ├─► Manual translation upload (import Fountain/FDX in target language)
       │
       └─► Dubbing script (separate workflow — adaptor rewrites for lip-sync)

Localized script version stored as a parallel AST branch
       │
       ▼
Dual-language export: original + translated side-by-side PDF
       │
       ▼
Distributor package: PDF + FDX + SRT (subtitle file) per language
```

### Dependencies

| Dependency | From | Why |
|-----------|------|-----|
| Script AST | Sprint 2 (P0) | Parallel AST branch per language version |
| AI Service | Sprint 6 (P0) | Translation via Sonnet (or self-hosted for data residency) |
| Import Service | Sprint 4 (P0) | Manual translation import uses existing FDX/Fountain parsers |
| Export Service | Sprint 4 (P0) | Localized PDF/FDX export |
| Bible Service | Sprint 5 (P0) | Character names and lore must be consistent across languages |

### Data Model (additions)

```sql
-- A localized script is a separate script_version with a language tag
ALTER TABLE script.script_versions ADD COLUMN language_code TEXT DEFAULT 'en';
ALTER TABLE script.script_versions ADD COLUMN source_version_id UUID REFERENCES script.script_versions(id);
-- source_version_id: if set, this is a translation of another version

CREATE TABLE script.translation_segments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_node_id   UUID NOT NULL REFERENCES script.ast_nodes(id),
  target_version_id UUID NOT NULL REFERENCES script.script_versions(id),
  language_code    TEXT NOT NULL,
  translated_text  TEXT NOT NULL,
  ai_generated     BOOLEAN NOT NULL DEFAULT FALSE,
  reviewed_by      UUID REFERENCES auth.users(id),
  reviewed_at      TIMESTAMPTZ,
  status           TEXT NOT NULL DEFAULT 'pending'  -- 'pending' | 'approved' | 'rejected'
);
```

### Pricing

Per-language package: included in Studio tier for up to 3 languages. Additional languages: $99/language/project. AI translation tokens charged against org quota (Sonnet rate). Dubbing adaptation: Studio tier only (specialized workflow).

---

## 4. Marketing & Pitch Generation

### What It Is

AI-assisted generation of marketing and pitch materials directly from the script and bible — loglines, one-pagers, synopsis documents, character bios for casting, lookbook copy, and pitch deck content.

Writers and producers currently create these manually, often re-reading the script to extract information that ScriptOS already has structured.

### User Flow

```
Producer opens "Marketing" panel for a script
       │
       ▼
Selects material type:
  - Logline (1–2 sentences)
  - Short synopsis (1 paragraph)
  - Long synopsis (1–2 pages, for distributors)
  - Character bios (from Bible + voice profiles)
  - One-pager (logline + synopsis + character breakdown + genre/tone)
  - Pitch deck copy (section-by-section content blocks)
       │
       ▼
AI assembles context: Script AST summary + Bible + character facts + genre
       │
       ▼
Sonnet generates draft → Writer edits → Finalizes
       │
       ▼
Export: DOCX, PDF, or copy to clipboard
```

### Dependencies

| Dependency | From | Why |
|-----------|------|-----|
| Script AST | Sprint 2 (P0) | Script content is the source material |
| Bible Service | Sprint 5 (P0) | Character facts, world rules, arcs feed pitch content |
| AI Service | Sprint 6 (P0) | Generation via Sonnet; prompt caching reuses Bible context |
| Breakdown Service | Sprint 6 (P0) | Genre, tone, element composition informs pitch positioning |

### AI Prompt Design

```
System (cacheable prefix):
  "You are a professional script analyst and marketing copywriter for the film/TV industry.
   You generate pitch materials from script data. Never claim the script is finished or 
   production-ready. Always frame as 'based on the script.'"

  [World rules]
  [Character facts + arcs]
  [Series tone + genre]

Variable suffix:
  "Generate a [logline / one-pager / synopsis] for the following script.
   Title: {title}
   Script summary: {ai-generated scene-by-scene summary from AST}
   Please write for a [studio executive / streaming buyer / film festival] audience."
```

### Scope Boundaries

**In scope:** Text-based marketing materials; DOCX/PDF export; Bible-grounded character descriptions.

**Out of scope (v1):** Image generation for lookbooks, trailer script generation, social media content, comps research (similar films).

---

## 5. Marketplace

### What It Is

A third-party ecosystem where developers, production companies, and tool vendors can publish plugins, templates, and integrations that extend ScriptOS — and optionally monetize them.

### What Gets Sold

| Category | Examples |
|---------|---------|
| Script templates | Genre-specific formatting templates, spec script templates, network-specific formats |
| Breakdown catalogs | Pre-populated element catalogs for specific production types (animation, documentary, commercial) |
| Export formatters | Custom PDF styles, network-specific delivery formats |
| AI prompt packs | Curated system prompts for specific genres or writing styles |
| Integrations | Connections to third-party tools (production accounting software, crew management) |
| Report templates | Custom call sheet layouts, budget report formats |

### Technical Architecture

```
Marketplace service (new P2 service)
  ├─ Plugin registry: published plugins with manifest + versioning
  ├─ Plugin sandbox: WASM-based execution environment (no native code)
  │    Plugins can: read AST, read Bible facts, write breakdown elements, generate exports
  │    Plugins cannot: make arbitrary network calls, access user credentials, write to AST directly
  ├─ Plugin store UI: browse, install, rate, review
  ├─ Revenue share: 70/30 split (publisher / ScriptOS) via Stripe Connect
  └─ Review queue: ScriptOS reviews all plugins before listing (security + quality)
```

### Plugin API Surface

```typescript
// packages/marketplace-sdk/ — published to npm for plugin developers

export interface PluginContext {
  // Read-only access to the current script
  getScript(): Promise<ScriptAST>;
  getSceneByNumber(sceneNumber: string): Promise<SceneNode | null>;

  // Read-only access to the Bible
  getCharacter(name: string): Promise<CharacterNode | null>;
  getWorldRules(): Promise<WorldRule[]>;

  // Write access (sandboxed)
  addBreakdownElement(element: BreakdownElementDraft): Promise<void>;
  exportToFormat(renderer: ExportRenderer): Promise<Buffer>;  // PDF/DOCX only

  // UI hooks
  registerPanel(panel: PanelDefinition): void;
  registerMenuItem(item: MenuItemDefinition): void;
  showNotification(message: string, level: 'info' | 'warning' | 'error'): void;
}
```

### Dependencies

This is the most complex P2 feature — requires mature P0/P1 before the plugin API surface is stable enough to publish to third-party developers.

| Dependency | From | Why |
|-----------|------|-----|
| All P0 services | Sprint 0–9 | Plugin API must not change; API surface needs to be stable |
| Breakdown Service | Sprint 6 (P0) | Primary write target for plugins |
| Script AST | Sprint 2 (P0) | Primary read source |
| Auth Service | Sprint 1 (P0) | Plugin-level permissions scoped under user/org RBAC |
| Watermark Service | Sprint 9 (P0) | Plugins that export must embed watermarks |

### Revenue Model

- **Free plugins**: listed for free; developer gets no revenue share
- **Paid plugins**: one-time purchase or monthly subscription; 70% to developer, 30% to ScriptOS
- **Enterprise plugins**: direct licensing deals for studio-specific tooling

### Scope Boundaries (Marketplace v1)

**In scope:** Plugin registry, WASM sandbox, read AST + Bible, write breakdown, custom export formatters, store UI, Stripe Connect payments, basic review queue.

**Out of scope (v1):** Plugin-generated AI prompts billed to org quota (too complex), marketplace analytics dashboard for developers, plugin versioning with backwards-compat guarantees, mobile plugin support.

---

## 6. Native Dailies Review Player

### What It Is

An in-app video player with frame-accurate annotations linked to script takes — replacing the Frame.io dependency for productions that want a fully integrated review experience. Includes AI-assisted take selection recommendations.

### Why After GA

Frame.io connector (Sprint 14) covers the immediate dailies review need. The native player differentiates in two ways that aren't possible with Frame.io:
1. **AI take selection**: ScriptOS knows the script, the character voice profiles, the continuity assertions, and the circled takes — it can recommend the "best" take with reasoning.
2. **Script-linked playback**: click a scene in the editor → jump to that scene's takes in the player. Click a take in the player → highlight the corresponding script lines.

These integrations are only possible in a native player.

### Dependencies

| Dependency | From | Why |
|-----------|------|-----|
| Supervisor Service | Sprint 10 (P0) | Take metadata, circled takes, deviation records |
| Continuity Service | Sprint 11 (P1) | Continuity assertions linked to takes |
| Frame.io connector | Sprint 14 (P1) | Native player replaces / supplements Frame.io, not precedes it |
| AI Service | Sprint 6 (P0) | Take selection recommendations use Sonnet |
| S3 (video storage) | Sprint 0 (P0) | Video files already in S3 via take upload pipeline |

### Technical Design

```
Video player: HLS.js (adaptive bitrate streaming from S3 via CloudFront)
Frame accuracy: WebCodecs API for frame-accurate seeking
Annotation layer: Canvas overlay keyed to frame number
Script sync: bidirectional — player highlights script lines; script click seeks player
AI take selection:
  - For each scene: load all takes + continuity assertions + deviation records
  - Call AI Service: "Given the script for this scene and these take records, 
    which take best serves the scene dramatically? Consider: deviation severity, 
    continuity consistency, notes."
  - Recommendation shown as a badge: "AI recommends Take 4" + 1-sentence reason
  - Writer/director accepts or overrides
```

### Scope Boundaries (Native Player v1)

**In scope:** HLS video player, frame annotations, take list, script sync, AI take selection recommendation, share link generation.

**Out of scope (v1):** Color grading previews, multi-cam switching, real-time remote viewing (Evercast covers that), offline playback in Tauri.

---

## 7. Evercast Connector

### What It Is

Evercast is a remote collaboration platform used by directors, producers, and editors to watch dailies together in real time — with synchronized playback, voice chat, and annotations. The ScriptOS connector links Evercast sessions to script takes, so review comments from an Evercast session land back in ScriptOS linked to the correct take and scene.

This is a narrower integration than Frame.io — Frame.io is async review; Evercast is live synchronous viewing sessions.

### User Flow

```
Director schedules a dailies review session
       │
       ▼
ScriptOS creates an Evercast room pre-loaded with selected takes
(takes organized by scene, in shoot order)
       │
       ▼
Participants join Evercast via link (outside ScriptOS)
Director watches takes, leaves comments (time-stamped, person-attributed)
       │
       ▼
ScriptOS webhook receives Evercast session end event
       │
       ▼
Comments imported → linked to takes in ScriptOS
→ appear in supervisor review comments panel
       │
       ▼
Supervisor addresses comments; resolves in ScriptOS
```

### Dependencies

Same as Frame.io connector (Sprint 14). Evercast connector is essentially a second implementation of the same webhook + comment ingestion pattern with a different provider API.

---

## P2 Priority Sequencing (Post-GA)

When GA user feedback comes in, prioritize P2 features using this framework:

| Signal | Implication |
|--------|------------|
| Beta users ask for table reads | Table Read TTS is Sprint 19–20 |
| International studios pilot at GA | Multilingual is Sprint 19–21 |
| Location managers frustrated by manual tracking | Location Scouting is Sprint 19–20 |
| Writers want pitch help | Marketing/Pitch is Sprint 19 (lowest build cost — reuses AI Service) |
| Third-party developers inquire about integration | Marketplace is Sprint 21–24 (highest build cost) |
| Frame.io contract/cost becomes an issue | Native Dailies Player accelerated |

**Recommended default P2 order (absent strong counter-signals from GA users):**

1. **Marketing & Pitch Generation** — lowest build cost (reuses AI Service, Bible, and AST); immediate writer value; no new infrastructure
2. **Table Read TTS** — high writer excitement; reuses Character Voice Registry; new infrastructure (TTS service + audio processing) but scoped
3. **Location Scouting** — unlocks significant Location Manager persona adoption; reuses Breakdown + Schedule data
4. **Multilingual Workflows** — high revenue potential for international; complex build; start planning in parallel with items 1–3
5. **Native Dailies Review Player** — replaces Frame.io dependency; AI take selection is a strong differentiator
6. **Evercast Connector** — narrow use case; low build cost once Frame.io connector pattern is established
7. **Marketplace** — highest complexity; requires all P0/P1 APIs to be stable; post-GA Year 2

---

## Sprint Estimates (Rough — Requires Dedicated Planning)

These are order-of-magnitude estimates for a full 8-engineer team. Each requires its own sprint-level planning session when prioritized.

| Feature | Estimated Sprints | New Services Required |
|---------|------------------|-----------------------|
| Marketing & Pitch Generation | 2 | None (AI Service extension) |
| Table Read TTS | 3 | `tts` service; audio processing worker |
| Location Scouting | 3 | `location` service; Google Maps API integration |
| Multilingual Workflows | 4–5 | `translation` service; parallel AST branching |
| Native Dailies Player | 4 | `player` frontend module; CloudFront CDN setup |
| Evercast Connector | 1 | None (webhook handler in Supervisor Service) |
| Marketplace | 6–8 | `marketplace` service; WASM plugin runtime; Stripe Connect |

---

## Decisions

**P2 features planned but not sprint-scheduled — deliberate.**
Sprint-level plans require stable APIs and user feedback to sequence correctly. Locking sprint tasks for P2 before GA user data exists produces plans that will be wrong. This document captures scope, dependencies, and technical direction so planning can start immediately after GA without re-deriving the architecture.

**All P2 features behind Unleash feature flags at GA.**
Flags allow selective enablement for pilot users and gradual rollout without code branches. Each P2 feature gets its own Unleash flag: `p2.table_read_tts`, `p2.location_scouting`, `p2.multilingual`, `p2.pitch_gen`, `p2.marketplace`, `p2.native_player`, `p2.evercast`.

**Marketing & Pitch Generation first by default.**
Lowest build cost (uses existing AI Service, Bible, and AST with no new infrastructure). Immediately useful to every writer and producer on the platform. Can ship as a minor feature in a Sprint 19 focused on AI depth rather than requiring a standalone sprint. Validates the AI-for-production-workflow use case before committing to larger P2 investments.

**Marketplace last by default.**
The plugin API surface must not change once third-party developers build against it. That requires all P0/P1 features to be stable — breaking changes to the AST, Bible, or Breakdown API would break plugins. Building Marketplace before the platform is stable would require costly backwards-compatibility shims. Year 2, post-GA, after platform APIs are locked.

**Actor voice cloning excluded from Table Read TTS v1.**
Cloning an actor's voice requires explicit informed consent, robust legal agreements, and careful model governance. The legal framework for this does not exist at ScriptOS's launch stage. Generic TTS voices (ElevenLabs voice library, OpenAI TTS presets) are sufficient for the table read use case. Actor voice cloning is a future consideration with dedicated legal review.
