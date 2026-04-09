# 09 — Post-Production & Editorial Round-Trip

## The Problem

Exporting to NLEs is straightforward. The hard part is receiving changes **back** from editorial — scene reorders, omissions, new takes — and mapping them to the script graph. Metadata evaporation (NLEs stripping custom metadata) is the primary risk.

## What Survives NLE Round-Trip

| Data | Survives? | Notes |
|------|-----------|-------|
| Timecodes | ✅ Always | Universal across all formats |
| Reel/tape names | ✅ Always | 8-char limit in EDL |
| Basic edit structure | ✅ Always | Cuts, dissolves, track layout |
| Custom metadata keys | ⚠️ Sometimes | NLE-dependent; often stripped |
| Script references | ❌ Usually lost | No NLE preserves arbitrary pipeline metadata |
| Effects/speed changes | ❌ Format-dependent | Complex constructs don't survive interchange |

## Interchange Format Comparison

| Format | Best For | Limitations |
|--------|----------|-------------|
| **OTIO** | Universal interchange; Resolve, Avid (via adapter), FCP | No built-in diff; complex effects lost |
| **AAF** | Avid workflows (required) | Binary format; Avid-specific extensions |
| **FCPXML** | Final Cut Pro workflows | Avid can't import; Premiere support limited |
| **EDL** | Universal compatibility, change lists | Single-track, 8-char reel names, no rich metadata |
| **ALE** | Avid metadata sync | Metadata only — no timeline structure |

## Bidirectional Architecture

```mermaid
flowchart TB
    subgraph "Export from ScriptOS"
        AST[Script AST] --> OTIO_EX[Generate OTIO\nwith scriptos metadata namespace]
        AST --> AAF_EX[Generate AAF\nfor Avid]
        AST --> EDL_EX[Generate EDL\nfor change lists]
        AST --> CORR[Populate Correlation DB]
    end

    subgraph "Correlation Database"
        CORR --> MAP[clip_name + source_timecode + reel_name\n→ scene_id + beat_id + take_id]
    end

    subgraph "NLE Work"
        EDITOR[Editor works in Avid/Premiere/Resolve]
    end

    subgraph "Re-Import to ScriptOS"
        EDITOR --> REIMPORT[Accept OTIO / AAF / EDL]
        REIMPORT --> CONVERT[Convert all to OTIO via adapters]
        CONVERT --> DIFF[Diff against previous export]
        DIFF --> MATCH_META[Match by scriptos metadata]
        MATCH_META -->|metadata present| OPS[Generate change operations]
        MATCH_META -->|metadata lost| MATCH_CORR[Fallback: match via correlation DB]
        MATCH_CORR --> OPS
        OPS --> REVIEW[Present changes for user review]
        REVIEW --> APPLY[Apply to script graph]
    end
```

## Correlation Database

The fallback matching system. Three identifiers survive **all** interchange formats, even legacy EDL:

```typescript
interface CorrelationRecord {
  // Matching keys (survive all formats)
  clip_name: string;
  source_timecode_in: string;
  reel_name: string;

  // ScriptOS references
  scene_id: string;
  beat_id: string;
  take_id: string;
  script_version: string;
  export_id: string;             // which export batch
  exported_at: string;
}
```

## Change Detection Operations

After diffing the re-imported OTIO against the previously exported version:

| Change Detected | Script Graph Operation |
|-----------------|----------------------|
| Clips from scene removed | Scene status → `omitted` |
| Clips reordered | Scene order updated |
| New clips from untracked source | Flagged for manual mapping |
| Source range changed | Beat-level timing adjustment |
| Different source clip for same scene | Take selection changed |

## Dailies Review Integration

```mermaid
flowchart LR
    subgraph "Dailies Sources"
        ONSET[On-set capture]
        LAB[Post-production lab]
    end

    subgraph "Review Surface"
        NATIVE[Native lightweight player]
        FRAMEIO[Frame.io connector]
        EVERCAST[Evercast connector]
    end

    subgraph "Comment Flow"
        RC[ReviewComment]
        RC --> FRAME[Frame number]
        RC --> TC[Timecode]
        RC --> RANGE[Duration range]
        RC --> REVIEWER[Reviewer identity]
        RC --> SCENE_REF[Scene/take reference]
    end

    subgraph "Downstream"
        RC --> MARKERS[Editorial marker packages]
        RC --> TURNOVER[Turnover packages]
    end

    ONSET --> NATIVE
    LAB --> FRAMEIO
    LAB --> EVERCAST
    NATIVE --> RC
    FRAMEIO --> RC
    EVERCAST --> RC
```

### ReviewComment Entity

```typescript
interface ReviewComment {
  id: string;
  frame: number;
  timecode: string;
  duration_range: { in: string; out: string } | null;
  reviewer_id: string;
  reviewer_name: string;
  comment_text: string;
  scene_ref: string | null;       // → Script AST scene ID
  take_ref: string | null;        // → TakeLog ID
  status: 'open' | 'addressed' | 'resolved';
  created_at: string;
}
```

## VFX Tracking Integration

Export structured metadata to VFX tracking systems:

| Integration | Format | Data Exchanged |
|-------------|--------|---------------|
| ShotGrid | REST API | Shots, tasks, versions, notes |
| ftrack | REST API | Shots, tasks, status, assignments |
| Custom pipeline | OTIO + JSON sidecar | Shot metadata, script references |

## Implementation Phases

| Phase | Deliverable | Dependencies |
|-------|-------------|-------------|
| Phase 1 | Export-only: OTIO + AAF + EDL + correlation DB | Script AST, Breakdown |
| Phase 2 | Import + diff + manual approval workflow | Phase 1, Workflow Orchestration |
| Phase 3 | Automated change propagation with webhook notifications | Phase 2, Event Bus |
| Phase 4 | NLE panel plugins (Premiere, Resolve) | Phase 3, client SDK |

## Open Questions

- [ ] Which NLEs to prioritize for round-trip: Avid? Premiere? Resolve?
- [ ] Dailies review: build native player or integrate Frame.io first?
- [ ] Change detection: file watching on shared storage vs manual export workflow?
- [ ] VFX integration: ShotGrid or ftrack first?
- [ ] OTIO adapter customization: contribute upstream or maintain fork?
