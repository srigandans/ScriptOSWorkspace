# 09 — Post-Production & Editorial Round-Trip

## The Problem

Exporting to NLEs is straightforward. The hard part is receiving changes **back** from editorial — scene reorders, omissions, new takes — and mapping them to the script graph. NLEs strip custom metadata; what survives round-trip is a small, stable set of identifiers (timecode, clip name, reel name). Everything else requires a correlation database and fallback matching strategy.

---

## 1. What Survives NLE Round-Trip

| Data | Survives? | Notes |
|------|-----------|-------|
| Timecodes (in/out) | ✅ Always | Universal across all formats |
| Reel / tape names | ✅ Always | 8-char limit in EDL; full in AAF/OTIO |
| Clip names | ✅ Usually | NLE-dependent naming conventions |
| Basic edit structure (cuts, track layout) | ✅ Always | Preserved in all formats |
| Custom metadata keys | ⚠️ Sometimes | OTIO `custom_metadata` namespace may survive; AAF custom extensions do not round-trip universally |
| Script / scene references | ❌ Usually lost | No NLE preserves arbitrary pipeline metadata natively |
| Effects, speed ramps, color grades | ❌ Format-dependent | Complex constructs don't survive interchange |
| Markers / comments | ⚠️ OTIO preserves markers | AAF marker round-trip is incomplete; EDL has none |

The correlation database exists to bridge this gap — it stores the mapping from stable NLE-visible identifiers to ScriptOS AST references, so the matching works even after metadata is stripped.

---

## 2. Interchange Format Strategy

| Format | NLE Targets | ScriptOS Usage | Notes |
|--------|------------|----------------|-------|
| **OTIO** | Resolve (native), Avid (adapter), FCP (adapter) | Primary export + re-import format | Open standard; custom metadata namespace; Python ecosystem for adapters |
| **AAF** | Avid Media Composer (required) | Export for Avid delivery | Binary format; parsed via `aaf` npm library or pyaaf; Avid-specific extensions |
| **FCPXML** | Final Cut Pro | v1.1 export only | Premiere support limited; FCP is v1.1 market |
| **EDL** | Universal (change lists only) | Legacy export / change list distribution | Single-track, 8-char reel names, no rich metadata |
| **ALE** | Avid bin metadata sync | Metadata re-import after AAF round-trip | Tab-delimited; carries `scriptos_scene_id` custom column |

**Primary round-trip path:** Export OTIO with `scriptos.*` custom metadata namespace → Avid converts to AAF internally → editor exports AAF → ScriptOS converts AAF→OTIO via adapter → diff against previous OTIO export.

---

## 3. Architecture

```
EXPORT PATH
───────────
Script AST (Postgres + CRDT)
         │
         ▼
  OTIOGenerator
  ├─ buildOTIOTimeline()        ← AST scenes → OTIO clips with scriptos metadata
  ├─ buildAAFSequence()         ← AAF for Avid delivery
  ├─ buildEDL()                 ← EDL for change lists
  └─ populateCorrelationDB()    ← store clip→scene mapping in correlation table
         │
         ▼
  S3 (export artifacts)    Postgres (editorial.correlation_records)


RE-IMPORT PATH
──────────────
Editor uploads OTIO / AAF / EDL
         │
         ▼
  FormatNormalizer             ← convert all formats to OTIO (via otio-python adapters)
         │
         ▼
  DiffEngine
  ├─ diffAgainstPreviousExport()
  └─ generateChangeOperations()
         │
         ▼
  MatchingEngine
  ├─ matchViaOTIOMetadata()     ← primary: scriptos.scene_id in custom_metadata
  └─ matchViaCorrelationDB()    ← fallback: timecode + reel_name + clip_name
         │
         ▼
  ChangeReviewWorkflow          ← Temporal saga: present changes, get approval, apply
         │
         ▼
  Script AST updated            ← scene omissions, reorders, take selections propagated
         │
         ▼
  NATS: editorial.changes.applied → Continuity Service, Breakdown Service, Notification
```

---

## 4. OTIO Export

### OTIO Timeline Structure

```
OTIOTimeline
  metadata:
    scriptos.project_id: "..."
    scriptos.script_version_id: "..."
    scriptos.export_id: "..."         ← correlation batch ID
    scriptos.exported_at: "..."
  tracks:
    - OTIOTrack (Video 1)
        - OTIOClip (Scene 1A INT. LAB - NIGHT)
            source_range: TimeRange(start=...)
            metadata:
              scriptos.scene_id: "uuid-v7"
              scriptos.scene_number: "1A"
              scriptos.int_ext: "INT"
              scriptos.location: "LAB"
              scriptos.time_of_day: "NIGHT"
              scriptos.page_count: 1.5
              scriptos.take_ids: ["take-uuid-1", "take-uuid-2"]
              scriptos.circled_take_id: "take-uuid-2"
              scriptos.breakdown_element_ids: ["elem-1", "elem-2"]
        - OTIOClip (Scene 2 EXT. ROOFTOP - DAY)
            ...
        - OTIOGap (omitted scenes produce gaps)
```

### TypeScript Implementation

```typescript
// services/supervisor/src/editorial/otio-generator.ts

import * as otio from 'opentimelineio';  // @scriptos/otio-node — Node.js bindings to OTIO C++ core

export interface OTIOExportInput {
  scriptVersionId: string;
  projectId: string;
  exportId: string;
  includeBreakdownRefs: boolean;
  includeCircledTakes: boolean;
}

export interface OTIOExportResult {
  s3Key: string;
  exportId: string;
  sceneCount: number;
  correlationRecordsWritten: number;
}

export async function generateOTIOExport(
  input: OTIOExportInput,
  db: DB,
  s3: S3Client,
): Promise<OTIOExportResult> {
  // Load scenes from Script AST (ordered by position)
  const scenes = await loadScenesForVersion(input.scriptVersionId, db);
  const circledTakes = input.includeCircledTakes
    ? await loadCircledTakes(scenes.map(s => s.id), db)
    : new Map<string, string>();

  const timeline = new otio.Timeline({
    name: `ScriptOS Export — version ${input.scriptVersionId}`,
    global_start_time: new otio.RationalTime(0, 24),
    metadata: {
      'scriptos.project_id': input.projectId,
      'scriptos.script_version_id': input.scriptVersionId,
      'scriptos.export_id': input.exportId,
      'scriptos.exported_at': new Date().toISOString(),
      'scriptos.schema_version': '1',
    },
  });

  const videoTrack = new otio.Track({ name: 'V1', kind: otio.TrackKind.Video });

  const correlationRecords: CorrelationRecord[] = [];
  let currentFrame = 0;

  for (const scene of scenes) {
    const durationFrames = Math.round((scene.page_count ?? 1) * 60); // rough: 1 page ≈ 1 min at 24fps
    const clipName = buildClipName(scene);

    const clip = new otio.Clip({
      name: clipName,
      source_range: new otio.TimeRange({
        start_time: new otio.RationalTime(currentFrame, 24),
        duration: new otio.RationalTime(durationFrames, 24),
      }),
      metadata: {
        'scriptos.scene_id': scene.id,
        'scriptos.scene_number': scene.scene_number,
        'scriptos.int_ext': scene.int_ext ?? '',
        'scriptos.location': scene.location_name ?? '',
        'scriptos.time_of_day': scene.time_of_day ?? '',
        'scriptos.page_count': scene.page_count?.toString() ?? '1',
        'scriptos.circled_take_id': circledTakes.get(scene.id) ?? '',
      },
    });

    videoTrack.append(clip);
    currentFrame += durationFrames;

    // Build correlation record — the fallback matching key
    correlationRecords.push({
      export_id: input.exportId,
      clip_name: clipName,
      source_timecode_in: frameToTimecode(currentFrame - durationFrames, 24),
      reel_name: buildReelName(scene),   // max 8 chars for EDL compat
      scene_id: scene.id,
      scene_number: scene.scene_number,
      script_version_id: input.scriptVersionId,
      project_id: input.projectId,
      exported_at: new Date().toISOString(),
    });
  }

  timeline.tracks.append(videoTrack);

  // Serialize to OTIO JSON
  const otioJson = otio.serialize(timeline, otio.SerializerVersion.OTIO_1_0_0);
  const s3Key = `editorial/exports/${input.projectId}/${input.exportId}/timeline.otio`;

  await s3.upload({
    Bucket: process.env.S3_BUCKET!,
    Key: s3Key,
    Body: otioJson,
    ContentType: 'application/json',
  });

  // Persist correlation records
  await writeCorrelationRecords(correlationRecords, db);

  return {
    s3Key,
    exportId: input.exportId,
    sceneCount: scenes.length,
    correlationRecordsWritten: correlationRecords.length,
  };
}

function buildClipName(scene: SceneRow): string {
  // Deterministic, human-readable, unique per scene
  // Pattern: {scene_number}_{INT_EXT}_{LOCATION}_{TOD}
  // e.g. "42A_INT_LAB_NIGHT"
  const parts = [
    scene.scene_number,
    scene.int_ext?.replace('./', '_').replace('.', '') ?? '',
    scene.location_name?.replace(/\s+/g, '_').toUpperCase().slice(0, 20) ?? 'UNKNOWN',
    scene.time_of_day?.replace(/\s+/g, '_').toUpperCase() ?? '',
  ].filter(Boolean);
  return parts.join('_');
}

function buildReelName(scene: SceneRow): string {
  // 8-char EDL limit: use scene number + hash suffix
  const base = scene.scene_number.replace(/\W/g, '').toUpperCase().slice(0, 5);
  const hash = scene.id.slice(0, 3).toUpperCase();
  return `${base}${hash}`.slice(0, 8);
}

function frameToTimecode(frame: number, fps: number): string {
  const h = Math.floor(frame / (fps * 3600));
  const m = Math.floor((frame % (fps * 3600)) / (fps * 60));
  const s = Math.floor((frame % (fps * 60)) / fps);
  const f = frame % fps;
  return `${pad(h)}:${pad(m)}:${pad(s)}:${pad(f)}`;
}
```

### AAF Export (Avid)

```typescript
// services/supervisor/src/editorial/aaf-generator.ts

import { AAFFile, Mob, SourceClip, Sequence, TimelineMobSlot } from '@scriptos/aaf-node';

export async function generateAAFExport(
  input: OTIOExportInput,
  db: DB,
  s3: S3Client,
): Promise<string> {
  const scenes = await loadScenesForVersion(input.scriptVersionId, db);
  const aaf = new AAFFile();

  // CompositionMob — the master timeline sequence
  const compMob = aaf.createCompositionMob(`ScriptOS_${input.scriptVersionId.slice(0, 8)}`);
  compMob.setCustomData('SCRIPTOS_VERSION_ID', input.scriptVersionId);
  compMob.setCustomData('SCRIPTOS_EXPORT_ID', input.exportId);

  const sequence = new Sequence({ mediaKind: 'picture', editRate: { numerator: 24, denominator: 1 } });

  for (const scene of scenes) {
    // Each scene becomes a SourceClip referencing a MasterMob
    const masterMob = aaf.createMasterMob(buildClipName(scene));
    masterMob.setCustomData('SCRIPTOS_SCENE_ID', scene.id);
    masterMob.setCustomData('SCRIPTOS_SCENE_NUMBER', scene.scene_number);

    const durationFrames = Math.round((scene.page_count ?? 1) * 1440); // 24fps * 60s

    const sourceClip = new SourceClip({
      length: durationFrames,
      startPosition: 0,
      sourceID: masterMob.mobID,
      sourceSlotID: 1,
    });

    sequence.appendComponent(sourceClip);
  }

  const slot = new TimelineMobSlot({ slotID: 1, name: 'V1', segment: sequence, editRate: { numerator: 24, denominator: 1 } });
  compMob.appendSlot(slot);
  aaf.addMob(compMob);

  const aafBuffer = aaf.toBuffer();
  const s3Key = `editorial/exports/${input.projectId}/${input.exportId}/timeline.aaf`;

  await s3.upload({
    Bucket: process.env.S3_BUCKET!,
    Key: s3Key,
    Body: aafBuffer,
    ContentType: 'application/octet-stream',
  });

  return s3Key;
}
```

---

## 5. Correlation Database

The correlation database is the fallback matching system. It maps the three identifiers that survive all NLE round-trips back to ScriptOS AST references.

### PostgreSQL DDL

```sql
-- Schema: editorial.*

CREATE TABLE editorial.correlation_records (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  export_id            UUID NOT NULL,              -- batch identifier
  project_id           UUID NOT NULL,
  script_version_id    UUID NOT NULL,
  -- Matching keys (survive NLE round-trip in all formats)
  clip_name            TEXT NOT NULL,
  source_timecode_in   TEXT NOT NULL,              -- HH:MM:SS:FF
  reel_name            TEXT NOT NULL,              -- max 8 chars for EDL compat
  -- ScriptOS references
  scene_id             UUID NOT NULL,
  scene_number         TEXT NOT NULL,
  circled_take_id      UUID,
  -- Audit
  exported_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  matched_at           TIMESTAMPTZ,               -- set when used in re-import match
  match_method         TEXT                       -- 'otio_metadata' | 'correlation_db' | 'manual'
);

CREATE INDEX ON editorial.correlation_records (export_id);
CREATE INDEX ON editorial.correlation_records (project_id, clip_name);
CREATE INDEX ON editorial.correlation_records (project_id, source_timecode_in, reel_name);
CREATE INDEX ON editorial.correlation_records (scene_id);

-- Export batch metadata
CREATE TABLE editorial.export_batches (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id           UUID NOT NULL,
  script_version_id    UUID NOT NULL,
  formats              TEXT[] NOT NULL,           -- ['otio', 'aaf', 'edl']
  exported_by          UUID NOT NULL REFERENCES auth.users(id),
  otio_s3_key          TEXT,
  aaf_s3_key           TEXT,
  edl_s3_key           TEXT,
  correlation_count    INTEGER NOT NULL DEFAULT 0,
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Re-import batches
CREATE TABLE editorial.reimport_batches (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id           UUID NOT NULL,
  source_export_id     UUID REFERENCES editorial.export_batches(id),
  uploaded_format      TEXT NOT NULL,              -- 'otio' | 'aaf' | 'edl'
  normalized_otio_key  TEXT NOT NULL,              -- S3 key to normalized OTIO
  uploaded_by          UUID NOT NULL REFERENCES auth.users(id),
  diff_summary         JSONB,
  workflow_id          TEXT,                       -- Temporal workflow ID for review saga
  status               TEXT NOT NULL DEFAULT 'pending',
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 6. Re-Import: Format Normalization

All incoming formats are first normalized to OTIO before diff and matching. This collapses the problem from N×N format handling to N→OTIO.

```typescript
// services/supervisor/src/editorial/format-normalizer.ts

import { execFile } from 'node:child_process';
import { promisify } from 'node:util';
import * as path from 'node:path';
import * as os from 'node:os';
import * as fs from 'node:fs/promises';

const execFileAsync = promisify(execFile);

export type ImportedFormat = 'otio' | 'aaf' | 'edl' | 'ale' | 'fcpxml';

export async function normalizeToOTIO(
  inputS3Key: string,
  format: ImportedFormat,
  s3: S3Client,
): Promise<{ normalizedS3Key: string }> {
  // Download the input file to a temp dir
  const tempDir = await fs.mkdtemp(path.join(os.tmpdir(), 'scriptos-editorial-'));
  const inputPath = path.join(tempDir, `input.${format}`);
  const outputPath = path.join(tempDir, 'output.otio');

  try {
    const fileBuffer = await s3.download({ Bucket: process.env.S3_BUCKET!, Key: inputS3Key });
    await fs.writeFile(inputPath, fileBuffer);

    if (format === 'otio') {
      // Already OTIO — just validate and re-serialize to ensure consistent format
      await execFileAsync('python3', [
        '-c',
        `import opentimelineio as otio; tl = otio.adapters.read_from_file('${inputPath}'); otio.adapters.write_to_file(tl, '${outputPath}')`,
      ]);
    } else {
      // Use OTIO Python adapter for conversion
      // otio-aaf-adapter, otio-edl-adapter, etc. are installed in the editorial worker image
      const adapterMap: Record<ImportedFormat, string> = {
        aaf: 'avid_aaf',
        edl: 'cmx_3600',
        fcpxml: 'fcpx_xml',
        ale: 'ale',
        otio: 'otio_json',
      };

      await execFileAsync('otioconvert', [
        '-i', inputPath,
        '-o', outputPath,
        '-O', adapterMap[format],
      ]);
    }

    // Upload normalized OTIO to S3
    const normalizedKey = inputS3Key.replace(/\.[^.]+$/, '.normalized.otio');
    const normalizedContent = await fs.readFile(outputPath);
    await s3.upload({
      Bucket: process.env.S3_BUCKET!,
      Key: normalizedKey,
      Body: normalizedContent,
      ContentType: 'application/json',
    });

    return { normalizedS3Key: normalizedKey };
  } finally {
    await fs.rm(tempDir, { recursive: true, force: true });
  }
}
```

---

## 7. Diff Engine

The diff engine compares the re-imported OTIO timeline against the previously exported OTIO timeline and generates a set of typed change operations.

```typescript
// services/supervisor/src/editorial/diff-engine.ts

import * as otio from 'opentimelineio';

export type ChangeOperation =
  | { type: 'scene_omitted'; sceneId: string; sceneNumber: string }
  | { type: 'scene_reordered'; sceneId: string; fromIndex: number; toIndex: number }
  | { type: 'take_selection_changed'; sceneId: string; oldTakeId: string | null; newTakeId: string | null }
  | { type: 'source_range_changed'; sceneId: string; oldRange: TimeRange; newRange: TimeRange }
  | { type: 'untracked_clip'; clipName: string; timecodeIn: string; reelName: string }
  | { type: 'scene_restored'; sceneId: string; sceneNumber: string };  // previously omitted, now present

export interface DiffResult {
  operations: ChangeOperation[];
  matchedClips: number;
  unmatchedClips: number;
  summary: string;
}

export async function diffTimelines(
  previousOTIOJson: string,
  reimportedOTIOJson: string,
  correlationRecords: CorrelationRecord[],
): Promise<DiffResult> {
  const previousTimeline = otio.deserialize(previousOTIOJson) as otio.Timeline;
  const reimportedTimeline = otio.deserialize(reimportedOTIOJson) as otio.Timeline;

  const previousClips = extractClips(previousTimeline);
  const reimportedClips = extractClips(reimportedTimeline);

  const operations: ChangeOperation[] = [];
  let matchedClips = 0;
  let unmatchedClips = 0;

  // Build index of previous clips by sceneId
  const prevBySceneId = new Map<string, OTIOClipInfo>();
  for (const [index, clip] of previousClips.entries()) {
    const sceneId = clip.metadata['scriptos.scene_id'];
    if (sceneId) prevBySceneId.set(sceneId, { ...clip, index });
  }

  // Build index of re-imported clips by sceneId (from metadata or correlation DB)
  const reimportedBySceneId = new Map<string, OTIOClipInfo>();
  for (const [index, clip] of reimportedClips.entries()) {
    const sceneId = resolveSceneId(clip, correlationRecords);
    if (sceneId) {
      reimportedBySceneId.set(sceneId, { ...clip, index });
      matchedClips++;
    } else {
      unmatchedClips++;
      operations.push({
        type: 'untracked_clip',
        clipName: clip.name,
        timecodeIn: clip.timecodeIn,
        reelName: clip.reelName,
      });
    }
  }

  // Detect omissions (in previous but not in re-imported)
  for (const [sceneId, prev] of prevBySceneId) {
    if (!reimportedBySceneId.has(sceneId)) {
      operations.push({
        type: 'scene_omitted',
        sceneId,
        sceneNumber: prev.metadata['scriptos.scene_number'] ?? '',
      });
    }
  }

  // Detect restorations (not in previous, but in re-imported — e.g., editor brought a cut scene back)
  for (const [sceneId, reimported] of reimportedBySceneId) {
    if (!prevBySceneId.has(sceneId)) {
      operations.push({
        type: 'scene_restored',
        sceneId,
        sceneNumber: reimported.metadata['scriptos.scene_number'] ?? '',
      });
    }
  }

  // Detect reorders
  const prevOrder = [...prevBySceneId.entries()].sort((a, b) => a[1].index - b[1].index).map(([id]) => id);
  const reimportOrder = [...reimportedBySceneId.entries()].sort((a, b) => a[1].index - b[1].index).map(([id]) => id);

  for (const sceneId of reimportOrder) {
    const prevIdx = prevOrder.indexOf(sceneId);
    const reimportIdx = reimportOrder.indexOf(sceneId);
    if (prevIdx !== -1 && prevIdx !== reimportIdx) {
      operations.push({ type: 'scene_reordered', sceneId, fromIndex: prevIdx, toIndex: reimportIdx });
    }
  }

  // Detect take selection changes
  for (const [sceneId, reimported] of reimportedBySceneId) {
    const prev = prevBySceneId.get(sceneId);
    if (!prev) continue;
    const prevTake = prev.metadata['scriptos.circled_take_id'] || null;
    const newTake = reimported.metadata['scriptos.circled_take_id'] || null;
    if (prevTake !== newTake) {
      operations.push({ type: 'take_selection_changed', sceneId, oldTakeId: prevTake, newTakeId: newTake });
    }
  }

  return {
    operations,
    matchedClips,
    unmatchedClips,
    summary: buildDiffSummary(operations),
  };
}

function resolveSceneId(clip: OTIOClipInfo, correlationRecords: CorrelationRecord[]): string | null {
  // Primary: OTIO custom metadata
  const metaSceneId = clip.metadata['scriptos.scene_id'];
  if (metaSceneId) return metaSceneId;

  // Fallback: correlation DB — match by timecode + reel_name + clip_name
  const record = correlationRecords.find(
    r =>
      r.source_timecode_in === clip.timecodeIn &&
      r.reel_name === clip.reelName,
  );
  if (record) return record.scene_id;

  // Second fallback: clip_name alone (less reliable but handles timecode drift)
  const byName = correlationRecords.find(r => r.clip_name === clip.name);
  return byName?.scene_id ?? null;
}

function buildDiffSummary(ops: ChangeOperation[]): string {
  const counts: Record<ChangeOperation['type'], number> = {
    scene_omitted: 0, scene_reordered: 0, take_selection_changed: 0,
    source_range_changed: 0, untracked_clip: 0, scene_restored: 0,
  };
  for (const op of ops) counts[op.type]++;

  const parts: string[] = [];
  if (counts.scene_omitted > 0) parts.push(`${counts.scene_omitted} scene(s) omitted`);
  if (counts.scene_restored > 0) parts.push(`${counts.scene_restored} scene(s) restored`);
  if (counts.scene_reordered > 0) parts.push(`${counts.scene_reordered} scene(s) reordered`);
  if (counts.take_selection_changed > 0) parts.push(`${counts.take_selection_changed} take selection(s) changed`);
  if (counts.untracked_clip > 0) parts.push(`${counts.untracked_clip} untracked clip(s)`);
  return parts.length > 0 ? parts.join(', ') : 'No changes detected';
}
```

---

## 8. Change Review Temporal Workflow

Editorial changes are never auto-applied. They go through a review workflow where the production coordinator or script supervisor approves or rejects each operation.

```typescript
// workers/revision-dist/src/workflows/editorial-reimport.ts

import { proxyActivities, defineSignal, defineQuery, setHandler, condition } from '@temporalio/workflow';
import type * as activities from '../activities/editorial';

const {
  loadCorrelationRecords,
  downloadAndNormalizeOTIO,
  runDiffEngine,
  presentChangesForReview,
  applySceneOmission,
  applySceneReorder,
  applyTakeSelectionChange,
  flagUntrackedClips,
  publishEditorialChangesApplied,
  markReimportFailed,
} = proxyActivities<typeof activities>({
  scheduleToCloseTimeout: '10 minutes',
  retry: { maximumAttempts: 3 },
});

export const editorialReviewSignal = defineSignal<[EditorialReviewDecision]>('editorial.review');
export const editorialStatusQuery = defineQuery<EditorialReimportStatus>('editorial.status');

export interface EditorialReimportInput {
  reimportBatchId: string;
  projectId: string;
  sourceExportId: string;
  uploadedOTIOKey: string;       // normalized OTIO S3 key
  reviewerId: string;
  previousExportOTIOKey: string;
}

export interface EditorialReviewDecision {
  approvedOperationIds: string[];
  rejectedOperationIds: string[];
}

export async function editorialReimportWorkflow(input: EditorialReimportInput): Promise<void> {
  let reviewDecision: EditorialReviewDecision | null = null;
  let status: EditorialReimportStatus = 'diffing';

  setHandler(editorialStatusQuery, () => status);
  setHandler(editorialReviewSignal, (d) => { reviewDecision = d; });

  try {
    // ── 1. Load correlation records for this export batch ─────────────────
    const correlationRecords = await loadCorrelationRecords({ exportId: input.sourceExportId });

    // ── 2. Diff ───────────────────────────────────────────────────────────
    const diffResult = await runDiffEngine({
      previousOTIOKey: input.previousExportOTIOKey,
      reimportedOTIOKey: input.uploadedOTIOKey,
      correlationRecords,
    });

    if (diffResult.operations.length === 0) {
      // No changes — nothing to review
      status = 'complete';
      return;
    }

    // ── 3. Present for review ─────────────────────────────────────────────
    status = 'awaiting_review';
    await presentChangesForReview({
      reimportBatchId: input.reimportBatchId,
      operations: diffResult.operations,
      summary: diffResult.summary,
      reviewerId: input.reviewerId,
      projectId: input.projectId,
    });

    // Wait up to 7 days for review (editorial workflows have longer lead times)
    await condition(() => reviewDecision !== null, '7 days');

    if (reviewDecision === null) {
      // Auto-cancel after 7 days
      await markReimportFailed({ reimportBatchId: input.reimportBatchId, reason: 'review_timeout' });
      return;
    }

    // ── 4. Apply approved operations ──────────────────────────────────────
    status = 'applying';
    const approvedOps = diffResult.operations.filter(
      op => reviewDecision!.approvedOperationIds.includes(operationId(op)),
    );

    // Apply in dependency order: omissions and restores first, then reorders, then takes
    const omissions = approvedOps.filter(op => op.type === 'scene_omitted');
    const restores = approvedOps.filter(op => op.type === 'scene_restored');
    const reorders = approvedOps.filter(op => op.type === 'scene_reordered');
    const takeChanges = approvedOps.filter(op => op.type === 'take_selection_changed');
    const untracked = diffResult.operations.filter(op => op.type === 'untracked_clip');

    await Promise.all([
      ...omissions.map(op => applySceneOmission({ ...op as any, projectId: input.projectId })),
      ...restores.map(op => applySceneReorder({ ...op as any, projectId: input.projectId })),
    ]);

    // Reorders only after omission/restoration is settled
    for (const op of reorders) {
      await applySceneReorder({ ...op as any, projectId: input.projectId });
    }

    await Promise.all(
      takeChanges.map(op => applyTakeSelectionChange({ ...op as any, projectId: input.projectId })),
    );

    // Flag untracked clips for manual mapping — never auto-apply
    if (untracked.length > 0) {
      await flagUntrackedClips({
        clips: untracked as any[],
        reimportBatchId: input.reimportBatchId,
        projectId: input.projectId,
      });
    }

    // ── 5. Publish NATS event ─────────────────────────────────────────────
    status = 'complete';
    await publishEditorialChangesApplied({
      reimportBatchId: input.reimportBatchId,
      projectId: input.projectId,
      operationsApplied: approvedOps.length,
      operationsRejected: reviewDecision.rejectedOperationIds.length,
    });

  } catch (err) {
    await markReimportFailed({ reimportBatchId: input.reimportBatchId, reason: (err as Error).message });
    throw err;
  }
}
```

---

## 9. Change Application Activities

```typescript
// workers/revision-dist/src/activities/editorial.ts

export async function applySceneOmission(input: {
  sceneId: string; sceneNumber: string; projectId: string;
}): Promise<void> {
  // Mark scene as omitted in the Script AST
  const scriptClient = createScriptServiceClient();
  await scriptClient.setSceneStatus({
    scene_id: input.sceneId,
    status: 'omitted',
    reason: 'editorial_reimport',
    project_id: input.projectId,
  });

  // Publish event for downstream — Breakdown Service will remove from active breakdown
  const nats = getNATSClient();
  await nats.publish('editorial.scene.omitted', {
    sceneId: input.sceneId,
    projectId: input.projectId,
    source: 'editorial_reimport',
  });
}

export async function applySceneReorder(input: {
  sceneId: string; toIndex: number; projectId: string;
}): Promise<void> {
  // Compute new fractional position for the scene in the AST
  const scriptClient = createScriptServiceClient();
  await scriptClient.reorderScene({
    scene_id: input.sceneId,
    target_index: input.toIndex,
    project_id: input.projectId,
    reason: 'editorial_reimport',
  });
}

export async function applyTakeSelectionChange(input: {
  sceneId: string;
  newTakeId: string | null;
  projectId: string;
}): Promise<void> {
  // Update the circled take on the supervisor scene record
  const supervisorClient = createSupervisorServiceClient();
  await supervisorClient.setCircledTake({
    scene_id: input.sceneId,
    take_id: input.newTakeId,
    source: 'editorial_reimport',
  });
}

export async function presentChangesForReview(input: {
  reimportBatchId: string;
  operations: ChangeOperation[];
  summary: string;
  reviewerId: string;
  projectId: string;
}): Promise<void> {
  const db = getDB();

  // Persist operations for UI rendering
  await db.query(sql`
    UPDATE editorial.reimport_batches
    SET diff_summary = ${JSON.stringify({ summary: input.summary, operations: input.operations })}
    WHERE id = ${input.reimportBatchId}
  `);

  // Notify reviewer
  const notificationClient = createNotificationServiceClient();
  await notificationClient.sendEditorialReviewRequest({
    reviewer_id: input.reviewerId,
    project_id: input.projectId,
    reimport_batch_id: input.reimportBatchId,
    summary: input.summary,
    review_url: `${process.env.APP_URL}/projects/${input.projectId}/editorial/review/${input.reimportBatchId}`,
  });
}

export async function publishEditorialChangesApplied(input: {
  reimportBatchId: string;
  projectId: string;
  operationsApplied: number;
  operationsRejected: number;
}): Promise<void> {
  const nats = getNATSClient();
  await nats.publish('editorial.changes.applied', {
    reimportBatchId: input.reimportBatchId,
    projectId: input.projectId,
    operationsApplied: input.operationsApplied,
    operationsRejected: input.operationsRejected,
  });
}
```

---

## 10. Frame.io Connector

Frame.io is the primary dailies review surface at launch. The connector listens to Frame.io webhooks for new comments and review decisions, then links them to ScriptOS take records.

### OAuth Integration

```typescript
// services/supervisor/src/frameio/oauth.ts

export const FRAMEIO_SCOPES = [
  'offline',          // refresh token
  'account.read',
  'asset.read',
  'asset.create',
  'comment.read',
  'comment.create',
  'review_link.read',
  'review_link.create',
];

export async function exchangeFrameIOCode(
  code: string,
  orgId: string,
  db: DB,
): Promise<void> {
  const response = await fetch('https://accounts.frame.io/auth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: process.env.FRAMEIO_CLIENT_ID,
      client_secret: process.env.FRAMEIO_CLIENT_SECRET,
      grant_type: 'authorization_code',
      code,
      redirect_uri: `${process.env.APP_URL}/integrations/frameio/callback`,
    }),
  });

  const tokens = await response.json();

  await db.query(sql`
    INSERT INTO integrations.frameio_credentials (org_id, access_token, refresh_token, expires_at)
    VALUES (${orgId}, ${tokens.access_token}, ${tokens.refresh_token}, NOW() + INTERVAL '${tokens.expires_in} seconds')
    ON CONFLICT (org_id) DO UPDATE
    SET access_token = EXCLUDED.access_token,
        refresh_token = EXCLUDED.refresh_token,
        expires_at = EXCLUDED.expires_at
  `);
}
```

### Webhook Handler

```typescript
// services/supervisor/src/frameio/webhook-handler.ts

export interface FrameIOWebhookEvent {
  type: 'comment.created' | 'asset.ready' | 'review_link.item_status_updated';
  data: Record<string, unknown>;
  account_id: string;
}

export async function handleFrameIOWebhook(
  event: FrameIOWebhookEvent,
  db: DB,
  nats: NATSClient,
): Promise<void> {
  switch (event.type) {
    case 'comment.created':
      await handleCommentCreated(event.data, db, nats);
      break;
    case 'asset.ready':
      await handleAssetReady(event.data, db, nats);
      break;
    case 'review_link.item_status_updated':
      await handleReviewDecision(event.data, db, nats);
      break;
  }
}

async function handleCommentCreated(data: Record<string, unknown>, db: DB, nats: NATSClient): Promise<void> {
  const assetId = data.asset_id as string;
  const commentText = (data.text as string) ?? '';
  const timecodeIn = data.timestamp as string | null;

  // Resolve asset → take via frameio_asset_links table
  const link = await db.queryOne<{ take_id: string; scene_id: string; project_id: string }>(sql`
    SELECT take_id, scene_id, project_id
    FROM integrations.frameio_asset_links
    WHERE frameio_asset_id = ${assetId}
  `);

  if (!link) {
    // Asset not linked to a ScriptOS take — store unresolved for manual linking
    await db.query(sql`
      INSERT INTO integrations.frameio_unresolved_comments (frameio_asset_id, comment_text, timecode_in, received_at)
      VALUES (${assetId}, ${commentText}, ${timecodeIn}, NOW())
    `);
    return;
  }

  // Create a ReviewComment record linked to the take
  const reviewComment: ReviewCommentRow = {
    id: crypto.randomUUID(),
    take_id: link.take_id,
    scene_id: link.scene_id,
    project_id: link.project_id,
    source: 'frameio',
    frameio_comment_id: data.id as string,
    comment_text: commentText,
    timecode_in: timecodeIn,
    reviewer_name: (data.author as any)?.name ?? 'Unknown',
    reviewer_id: null,  // mapped from Frame.io user to ScriptOS user if possible
    status: 'open',
    created_at: new Date().toISOString(),
  };

  await db.query(sql`
    INSERT INTO supervisor.review_comments
      (id, take_id, scene_id, project_id, source, frameio_comment_id, comment_text, timecode_in, reviewer_name, status, created_at)
    VALUES
      (${reviewComment.id}, ${reviewComment.take_id}, ${reviewComment.scene_id}, ${reviewComment.project_id},
       ${reviewComment.source}, ${reviewComment.frameio_comment_id}, ${reviewComment.comment_text},
       ${reviewComment.timecode_in}, ${reviewComment.reviewer_name}, ${reviewComment.status}, ${reviewComment.created_at})
  `);

  // Notify supervisor
  await nats.publish('supervisor.review_comment.created', {
    commentId: reviewComment.id,
    takeId: link.take_id,
    sceneId: link.scene_id,
    projectId: link.project_id,
    source: 'frameio',
  });
}

async function handleReviewDecision(data: Record<string, unknown>, db: DB, nats: NATSClient): Promise<void> {
  // status: 'approved' | 'needs_review' | 'in_review'
  const status = data.status as string;
  const assetId = data.asset_id as string;

  const link = await db.queryOne<{ take_id: string; scene_id: string }>(sql`
    SELECT take_id, scene_id FROM integrations.frameio_asset_links WHERE frameio_asset_id = ${assetId}
  `);

  if (!link) return;

  if (status === 'approved') {
    // Mark take as director-approved in supervisor
    await db.query(sql`
      UPDATE supervisor.takes SET director_approved = TRUE, updated_at = NOW()
      WHERE id = ${link.take_id}
    `);

    await nats.publish('supervisor.take.director_approved', {
      takeId: link.take_id,
      sceneId: link.scene_id,
      source: 'frameio_review',
    });
  }
}
```

### Asset Linking (Upload → Link to Take)

```typescript
// services/supervisor/src/frameio/asset-linker.ts

export async function uploadTakeToFrameIO(
  takeId: string,
  videoS3Key: string,
  projectId: string,
  db: DB,
  s3: S3Client,
): Promise<string> {
  // Get the Frame.io project ID for this ScriptOS project
  const config = await db.queryOne<{ frameio_project_id: string; access_token: string }>(sql`
    SELECT fip.frameio_project_id, fc.access_token
    FROM integrations.frameio_project_links fip
    JOIN integrations.frameio_credentials fc ON fc.org_id = fip.org_id
    WHERE fip.project_id = ${projectId}
  `);

  if (!config) throw new Error(`No Frame.io project linked for project ${projectId}`);

  // Create asset placeholder in Frame.io
  const createResponse = await fetch(
    `https://api.frame.io/v2/assets/${config.frameio_project_id}/children`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${config.access_token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        name: `take_${takeId}.mp4`,
        type: 'file',
        filetype: 'video/mp4',
      }),
    },
  );

  const asset = await createResponse.json();

  // Upload via Frame.io presigned URL
  const videoBuffer = await s3.download({ Bucket: process.env.S3_BUCKET!, Key: videoS3Key });
  await fetch(asset.upload_urls[0], { method: 'PUT', body: videoBuffer });

  // Store asset link
  await db.query(sql`
    INSERT INTO integrations.frameio_asset_links (frameio_asset_id, take_id, project_id, created_at)
    VALUES (${asset.id}, ${takeId}, ${projectId}, NOW())
  `);

  return asset.id;
}
```

---

## 11. Review Comments PostgreSQL Schema

```sql
-- Schema: supervisor.*

CREATE TABLE supervisor.review_comments (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  take_id           UUID REFERENCES supervisor.takes(id),
  scene_id          UUID NOT NULL REFERENCES supervisor.scenes(id),
  project_id        UUID NOT NULL,
  source            TEXT NOT NULL,                -- 'frameio' | 'native' | 'manual'
  frameio_comment_id TEXT,                        -- Frame.io comment ID for deduplication
  comment_text      TEXT NOT NULL,
  timecode_in       TEXT,                          -- HH:MM:SS:FF within the clip
  timecode_out      TEXT,
  reviewer_name     TEXT NOT NULL,
  reviewer_id       UUID REFERENCES auth.users(id),
  status            TEXT NOT NULL DEFAULT 'open', -- 'open' | 'addressed' | 'resolved'
  resolved_at       TIMESTAMPTZ,
  resolved_by       UUID REFERENCES auth.users(id),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ON supervisor.review_comments (scene_id, status);
CREATE INDEX ON supervisor.review_comments (take_id);
CREATE INDEX ON supervisor.review_comments (project_id, created_at);
CREATE UNIQUE INDEX ON supervisor.review_comments (frameio_comment_id) WHERE frameio_comment_id IS NOT NULL;

-- Frame.io integration tables
CREATE TABLE integrations.frameio_credentials (
  org_id        UUID PRIMARY KEY REFERENCES auth.organizations(id),
  access_token  TEXT NOT NULL,
  refresh_token TEXT NOT NULL,
  expires_at    TIMESTAMPTZ NOT NULL,
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE integrations.frameio_project_links (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id               UUID NOT NULL REFERENCES auth.organizations(id),
  project_id           UUID NOT NULL,
  frameio_project_id   TEXT NOT NULL,
  frameio_project_name TEXT,
  linked_at            TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE integrations.frameio_asset_links (
  frameio_asset_id TEXT PRIMARY KEY,
  take_id          UUID REFERENCES supervisor.takes(id),
  project_id       UUID NOT NULL,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE integrations.frameio_unresolved_comments (
  id                TEXT PRIMARY KEY DEFAULT gen_random_uuid()::TEXT,
  frameio_asset_id  TEXT NOT NULL,
  comment_text      TEXT NOT NULL,
  timecode_in       TEXT,
  received_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 12. VFX Tracking Integration (ShotGrid)

```typescript
// services/supervisor/src/vfx/shotgrid-sync.ts

const SHOTGRID_API_BASE = 'https://studio.shotgunstudio.com/api/v1';

export async function syncShotToShotGrid(
  sceneId: string,
  projectId: string,
  shotgridProjectId: number,
  db: DB,
): Promise<void> {
  const scene = await loadScene(sceneId, db);
  const apiKey = await loadShotGridAPIKey(projectId, db);

  // Create or update Shot entity in ShotGrid
  const existingShot = await findExistingShotGridShot(scene.scene_number, shotgridProjectId, apiKey);

  if (existingShot) {
    await updateShotGridShot(existingShot.id, scene, apiKey);
  } else {
    const shot = await createShotGridShot({
      projectId: shotgridProjectId,
      code: scene.scene_number,
      description: scene.synopsis ?? '',
      shotType: `${scene.int_ext ?? ''} ${scene.location_name ?? ''}`.trim(),
      pageCount: scene.page_count,
      scriptOSSceneId: scene.id,   // custom field: sg_scriptos_scene_id
      apiKey,
    });

    await db.query(sql`
      INSERT INTO integrations.shotgrid_shot_links (scene_id, shotgrid_shot_id, project_id)
      VALUES (${sceneId}, ${shot.id}, ${projectId})
      ON CONFLICT (scene_id) DO UPDATE SET shotgrid_shot_id = EXCLUDED.shotgrid_shot_id
    `);
  }
}

async function createShotGridShot(input: {
  projectId: number;
  code: string;
  description: string;
  shotType: string;
  pageCount: number | null;
  scriptOSSceneId: string;
  apiKey: string;
}): Promise<{ id: number }> {
  const response = await fetch(`${SHOTGRID_API_BASE}/entity/Shot`, {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${input.apiKey}`,
      'Content-Type': 'application/vnd+shotgun.api3_array+json',
    },
    body: JSON.stringify({
      data: {
        project: { type: 'Project', id: input.projectId },
        code: input.code,
        description: input.description,
        sg_shot_type: input.shotType,
        sg_cut_duration: input.pageCount ? Math.round(input.pageCount * 60) : null,
        sg_scriptos_scene_id: input.scriptOSSceneId,
      },
    }),
  });

  const result = await response.json();
  return { id: result.data.id };
}
```

---

## 13. Testing Strategy

### Unit Tests

```typescript
// services/supervisor/src/__tests__/editorial/diff-engine.test.ts

describe('diffTimelines', () => {
  it('detects scene omission when clip absent from re-imported timeline', async () => {
    const previous = buildTestOTIO([
      { sceneId: 'scene-1', sceneNumber: '1' },
      { sceneId: 'scene-2', sceneNumber: '2' },
    ]);
    const reimported = buildTestOTIO([
      { sceneId: 'scene-1', sceneNumber: '1' },
      // scene-2 omitted
    ]);

    const result = await diffTimelines(previous, reimported, []);
    const omission = result.operations.find(op => op.type === 'scene_omitted');
    expect(omission).toBeDefined();
    expect((omission as any).sceneId).toBe('scene-2');
  });

  it('falls back to correlation DB when OTIO metadata is absent', async () => {
    const previous = buildTestOTIO([{ sceneId: 'scene-1', sceneNumber: '1' }]);
    // Re-imported has no scriptos metadata
    const reimported = buildOTIOWithoutMetadata([{ clipName: '1_INT_LAB_NIGHT', timecodeIn: '00:00:00:00', reelName: '1LABNI' }]);

    const correlationRecords: CorrelationRecord[] = [
      { scene_id: 'scene-1', clip_name: '1_INT_LAB_NIGHT', source_timecode_in: '00:00:00:00', reel_name: '1LABNI', export_id: 'exp-1', script_version_id: 'v-1', project_id: 'proj-1', exported_at: new Date().toISOString(), scene_number: '1' },
    ];

    const result = await diffTimelines(previous, reimported, correlationRecords);
    expect(result.matchedClips).toBe(1);
    expect(result.unmatchedClips).toBe(0);
    expect(result.operations).toHaveLength(0); // same scene, no changes
  });

  it('detects reorder when clips appear in different sequence', async () => {
    const previous = buildTestOTIO([
      { sceneId: 'scene-1', sceneNumber: '1' },
      { sceneId: 'scene-2', sceneNumber: '2' },
      { sceneId: 'scene-3', sceneNumber: '3' },
    ]);
    const reimported = buildTestOTIO([
      { sceneId: 'scene-1', sceneNumber: '1' },
      { sceneId: 'scene-3', sceneNumber: '3' },  // reordered
      { sceneId: 'scene-2', sceneNumber: '2' },
    ]);

    const result = await diffTimelines(previous, reimported, []);
    const reorders = result.operations.filter(op => op.type === 'scene_reordered');
    expect(reorders.length).toBeGreaterThan(0);
  });

  it('flags untracked clips not in correlation DB', async () => {
    const previous = buildTestOTIO([]);
    const reimported = buildOTIOWithoutMetadata([
      { clipName: 'PICKUP_SHOT', timecodeIn: '01:00:00:00', reelName: 'PICKUP' },
    ]);

    const result = await diffTimelines(previous, reimported, []);
    expect(result.unmatchedClips).toBe(1);
    const untracked = result.operations.find(op => op.type === 'untracked_clip');
    expect((untracked as any).clipName).toBe('PICKUP_SHOT');
  });
});
```

### Integration Tests

```typescript
// services/supervisor/src/__tests__/integration/editorial-roundtrip.test.ts

describe('full editorial round-trip', () => {
  it('export → NLE simulation → re-import → review → apply', async () => {
    // 1. Create test project with scenes
    const { projectId, scriptVersionId } = await setupTestProject();

    // 2. Export OTIO
    const exportResult = await generateOTIOExport({ scriptVersionId, projectId, exportId: uuid(), includeBreakdownRefs: false, includeCircledTakes: false }, testDB, testS3);
    expect(exportResult.sceneCount).toBeGreaterThan(0);
    expect(exportResult.correlationRecordsWritten).toBe(exportResult.sceneCount);

    // 3. Simulate NLE: omit one scene, reorder two others
    const reimportedOTIO = await simulateNLEEditing(exportResult.s3Key, {
      omitSceneIndex: 1,
      swapSceneIndices: [2, 3],
    });
    const reimportKey = await uploadToTestS3(reimportedOTIO);

    // 4. Start reimport workflow
    const { workflowId } = await startEditorialReimportWorkflow({
      projectId,
      sourceExportId: exportResult.exportId,
      uploadedOTIOKey: reimportKey,
      previousExportOTIOKey: exportResult.s3Key,
      reviewerId: 'test-coordinator',
    });

    // 5. Get diff
    await waitForWorkflowStatus(workflowId, 'awaiting_review');
    const batch = await testDB.queryOne(sql`SELECT diff_summary FROM editorial.reimport_batches WHERE workflow_id = ${workflowId}`);
    const ops = batch.diff_summary.operations;

    expect(ops.some(op => op.type === 'scene_omitted')).toBe(true);
    expect(ops.some(op => op.type === 'scene_reordered')).toBe(true);

    // 6. Approve all operations
    await signalWorkflow(workflowId, editorialReviewSignal, {
      approvedOperationIds: ops.map(op => operationId(op)),
      rejectedOperationIds: [],
    });

    await waitForWorkflowStatus(workflowId, 'complete');

    // 7. Verify AST updated
    const omittedScene = await testDB.queryOne(sql`
      SELECT status FROM script.ast_nodes WHERE id = ${ops.find(op => op.type === 'scene_omitted').sceneId}
    `);
    expect(omittedScene.status).toBe('omitted');
  });
});
```

---

## Decisions

**NLE priority — Avid (AAF) first, DaVinci Resolve (OTIO) second, Premiere (FCPXML) in v1.1. See ADR-022.**
High-budget TV drama and feature film productions — the primary ScriptOS market — run on Avid Media Composer. Avid requires AAF specifically; OTIO alone is insufficient for professional Avid workflows. DaVinci Resolve has native OTIO support and is the easiest second integration. Premiere Pro is common in digital/streaming but is lower priority than Avid and Resolve for the target market at launch.

**Dailies review — Frame.io connector first (v1); native player in v1.1. See ADR-023.**
Frame.io is the industry standard for dailies review. Productions already have Frame.io workflows and accounts. Building a native player at launch competes with a well-established tool without adding differentiation. The Frame.io connector (OAuth, webhook for comments, take-reference linking) reaches market faster. A native lightweight player with script-linked annotations and AI take-selection suggestions becomes worthwhile in v1.1 when ScriptOS has features Frame.io doesn't.

**Re-import matching — dual strategy: OTIO metadata first, correlation DB fallback.**
OTIO `custom_metadata` survives the export→NLE→re-import cycle when the editor stays in OTIO-native NLEs (Resolve, FCP). AAF round-trips and EDL re-imports strip custom metadata entirely. The correlation DB guarantees matching works in all cases. Storing three stable identifiers (clip_name + timecode_in + reel_name) is sufficient for unambiguous matching when the script hasn't been re-numbered between export and re-import.

**Change application — never auto-apply; always require coordinator review.**
NLE edits can reflect director decisions (scene omissions, reorders) that are not yet reflected in the production document. Auto-applying changes could corrupt the script graph if the editor accidentally omitted a scene or if there's a metadata mismatch. A review step lets the production coordinator confirm each operation before it's committed. The review workflow has a 7-day timeout — long enough for real production cycles. After timeout, the batch is cancelled (not auto-applied).

**Untracked clips — flag for manual mapping, never auto-apply.**
Clips not matched by metadata or correlation DB may be pickup shots, inserts, or new coverage filmed after the script was exported. They cannot be automatically assigned to a scene without human judgment. They are flagged in a persistent queue for the production coordinator to resolve. They are never silently discarded.

**OTIO adapter — contribute upstream; fork only if upstream is too slow.**
Maintaining an OTIO adapter fork creates ongoing merge overhead against an actively maintained project. The `scriptos.*` custom metadata namespace uses OTIO's built-in custom metadata mechanism — no fork needed. Any fixes to existing adapters (AAF, FCPXML) are contributed upstream first. A fork is created only if the contribution process can't meet a specific release deadline.

**VFX integration — ShotGrid (Autodesk Flow Production Tracking) first; ftrack in v1.1.**
ShotGrid is the dominant VFX tracking platform in major productions (VFX-heavy features and TV, Autodesk ownership, broad studio adoption). ftrack is strong but more common in mid-sized studios. Both use REST APIs — the integration hub abstracts the difference. ShotGrid first covers the highest-value clients.

**Format normalization — all formats converted to OTIO before diff.**
Diffing N input formats against M previously-exported formats requires N×M comparison implementations. Normalizing all inputs to OTIO collapses this to a single diff implementation. The normalization step uses OTIO's standard Python adapters (`otio-aaf-adapter`, `otio-edl-adapter`) run as a subprocess in the editorial worker container. This is slightly slower than native parsing but correct and maintainable.
