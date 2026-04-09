# 04 — CRDT & Collaboration — Technical Deep-Dive

> **Status:** FINALIZED — v1.0 (2026-04-09)
> Step 3 deep-dive. Supersedes the prior architectural overview. All decisions from the overview are preserved and expanded into production specifications.

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Library Decision](#2-library-decision)
3. [Loro Document Model](#3-loro-document-model)
4. [ProseMirror ↔ Loro Binding](#4-prosemirror--loro-binding)
5. [WebSocket Sync Protocol](#5-websocket-sync-protocol)
6. [Awareness Protocol](#6-awareness-protocol)
7. [Structural Operations — Split, Merge, Move](#7-structural-operations--split-merge-move)
8. [Checkpoint Protocol (CRDT → Postgres)](#8-checkpoint-protocol-crdt--postgres)
9. [Compaction Algorithm](#9-compaction-algorithm)
10. [Sync Infrastructure — y-redis / Hocuspocus](#10-sync-infrastructure--y-redis--hocuspocus)
11. [Yjs Fallback Specification](#11-yjs-fallback-specification)
12. [Semantic Validation Layer](#12-semantic-validation-layer)
13. [Offline Operation (Tauri On-Set)](#13-offline-operation-tauri-on-set)
14. [Error Handling and Recovery](#14-error-handling-and-recovery)
15. [Testing Strategy](#15-testing-strategy)
16. [Decisions](#16-decisions)

---

## 1. Problem Statement

ScriptOS requires real-time collaboration on two structurally different operations:

**Text editing** — character-level insertions and deletions inside action blocks, dialogue lines, scene headings, and parentheticals. Must work offline (on-set Script Supervisor). Must converge without central coordination.

**Structural editing** — moving scenes between acts/sequences, splitting a scene into two, merging adjacent scenes, reordering acts, creating/deleting acts and sequences. These operations change the tree topology. Naive CRDT delete+insert causes node duplication on concurrent moves.

The tree move problem is the core challenge. See the diagram below.

```
Concurrent scene move with naive delete+insert CRDT:

State:  Act 1 [Scene 5]    Act 2 []    Act 3 []

Writer A: Move Scene 5 → Act 2   (delete from Act 1, insert at Act 2)
Writer B: Move Scene 5 → Act 3   (delete from Act 1, insert at Act 3)

After merge:
  Writer A deleted from Act 1, inserted at Act 2
  Writer B deleted from Act 1, inserted at Act 3
  Both inserts survive → Scene 5 appears in BOTH Act 2 AND Act 3 ❌

With Loro MovableTree (Kleppmann 2021):
  Move is a single atomic operation with Lamport timestamp
  Higher timestamp wins → Scene 5 ends up in exactly one location ✅
```

---

## 2. Library Decision

**Loro is primary.** POC gate at Week 6. If the ProseMirror binding POC reveals a blocking issue, pivot to Yjs fallback (§11). The decision criteria:

| Criterion | Pass | Fail |
|-----------|------|------|
| Binding round-trip | PM transaction → Loro op → PM transaction reconstructs identical doc | Any data loss or corruption |
| Concurrent text convergence | 3 concurrent writers, 1000 ops, all reach same state | Any divergence |
| Structural move convergence | Concurrent moves to different parents resolve without duplication | Any duplication |
| Performance | < 5ms latency added by binding on 90th percentile keypress | > 10ms added latency |
| Memory | < 20MB Loro state for a 120-page feature screenplay | > 50MB |

The Yjs fallback is fully specified in §11 so the pivot, if needed, is a swap not a redesign.

---

## 3. Loro Document Model

### One LoroDoc per Script

Each script (feature or episode) has exactly one `LoroDoc`. The document ID is the script UUID from Postgres. The same `LoroDoc` contains both the structural tree and all text content — they share a single version vector and can be synced atomically.

```typescript
import { LoroDoc, LoroTree, LoroText, LoroMap } from 'loro-crdt';

/**
 * The single Loro document for a script.
 * Created fresh on first edit; loaded from crdt_snapshot on reconnect.
 */
function createScriptDoc(scriptId: string): LoroDoc {
  const doc = new LoroDoc();
  // Peer ID must be unique per client session and stable within a session.
  // Derived from: userId + sessionId (not just userId, to handle multi-tab).
  doc.setPeerId(derivePeerId(scriptId));
  return doc;
}

function derivePeerId(scriptId: string): bigint {
  // XOR of script UUID bytes with a random session salt.
  // Ensures uniqueness across concurrent sessions by the same user.
  const sessionSalt = crypto.getRandomValues(new Uint8Array(8));
  // ... derive bigint from hash of scriptId + sessionSalt
}
```

### Structural Tree (LoroTree)

The AST node hierarchy lives in a single `LoroTree` keyed as `"structure"` within the doc. Every AST node has a corresponding `LoroTree` node with a matching UUID.

```typescript
/**
 * Bootstrap the LoroTree from the Postgres AST on first load.
 * Subsequent sessions load from crdt_snapshot — this is only for
 * scripts with no prior CRDT state (new scripts or imported scripts).
 */
function bootstrapStructureFromAST(
  doc: LoroDoc,
  nodes: ASTNode[],  // from Postgres ast_nodes, ordered by position
): void {
  const tree = doc.getTree('structure');

  // Sort nodes: parents before children (topological order)
  const ordered = topologicalSort(nodes);

  for (const node of ordered) {
    if (node.parent_id === null) {
      // Root node (project) — LoroTree has its own virtual root
      const treeNode = tree.createNode();
      treeNode.data.set('ast_id', node.id);
      treeNode.data.set('type', node.type);
      treeNode.data.set('position', node.position);
      treeNode.data.set('metadata', JSON.stringify(node.metadata));
    } else {
      const parentTreeNode = findTreeNodeByAstId(tree, node.parent_id);
      const treeNode = tree.createNode(parentTreeNode.id);
      treeNode.data.set('ast_id', node.id);
      treeNode.data.set('type', node.type);
      treeNode.data.set('position', node.position);
      treeNode.data.set('metadata', JSON.stringify(node.metadata));
    }
  }
}
```

### Text Content (LoroText)

Each leaf AST node has a corresponding `LoroText` keyed by the node's UUID. Leaf nodes are: `scene_heading`, `action`, `character_name`, `parenthetical`, `dialogue`, `transition`, `section_break`, `synopsis`, `note`.

```typescript
/**
 * Get or create the LoroText for a leaf node.
 * Key format: "text:{node_uuid}"
 * This key is stable — it is the node UUID from Postgres, never changes.
 */
function getNodeText(doc: LoroDoc, nodeId: string): LoroText {
  return doc.getText(`text:${nodeId}`);
}

/**
 * Bootstrap text content for a leaf node from its Postgres content_snapshot.
 * Called once per node on first CRDT session.
 */
function bootstrapNodeText(
  doc: LoroDoc,
  nodeId: string,
  contentSnapshot: string,
): void {
  const text = getNodeText(doc, nodeId);
  if (text.length === 0 && contentSnapshot.length > 0) {
    text.insert(0, contentSnapshot);
  }
}
```

### Node Metadata (LoroMap)

Per-node metadata (scene int_ext, location_ref, story_day_ref, etc.) lives in a `LoroMap` keyed by `"meta:{node_uuid}"`. Only the fields that need real-time collaborative editing live in the CRDT map — the full metadata is in Postgres. The CRDT metadata map is for fields that writers might edit simultaneously (e.g., two writers both assigning a scene to a story day).

```typescript
type CRDTMetadataFields = {
  // Scene-level fields that multiple writers might change concurrently
  scene_number?: string | null;
  story_day_ref?: string | null;
  time_of_day?: string | null;
  revision_color?: string | null;
  omitted?: boolean;
};

function getNodeMeta(doc: LoroDoc, nodeId: string): LoroMap {
  return doc.getMap(`meta:${nodeId}`);
}
```

### Complete Document Key Space

```
LoroDoc (scriptId)
│
├── tree: "structure"
│   ├── node (ast_id: uuid, type: 'script', position: 0, metadata: {...})
│   │   ├── node (ast_id: uuid, type: 'act', position: 1.0, ...)
│   │   │   ├── node (ast_id: uuid, type: 'scene', position: 1.0, ...)
│   │   │   │   ├── node (ast_id: uuid, type: 'scene_heading', ...)
│   │   │   │   ├── node (ast_id: uuid, type: 'action', ...)
│   │   │   │   └── node (ast_id: uuid, type: 'dialogue_group', ...)
│   │   │   │       ├── node (ast_id: uuid, type: 'character_name', ...)
│   │   │   │       └── node (ast_id: uuid, type: 'dialogue', ...)
│   │   │   └── ...
│   │   └── ...
│   └── ...
│
├── text: "text:{uuid}" → LoroText  (one per leaf node)
│   ├── "text:{scene_heading_uuid}"    → "INT. COFFEE SHOP - NIGHT"
│   ├── "text:{action_uuid}"           → "Jake enters, scanning the room..."
│   ├── "text:{character_name_uuid}"   → "JAKE"
│   ├── "text:{dialogue_uuid}"         → "I need to find her."
│   └── ...
│
└── map: "meta:{uuid}" → LoroMap  (one per node with collaborative metadata)
    ├── "meta:{scene_uuid}"   → { story_day_ref: "...", time_of_day: "NIGHT", ... }
    └── ...
```

### Version Vectors and Frontiers

Loro uses a **frontier** (set of latest operation IDs per peer) rather than a monotonic version counter. This is critical for the sync protocol.

```typescript
// Export the current frontier (what this peer has seen)
const frontier: Uint8Array = doc.version();

// Export all ops since a given frontier (what to send to a peer)
const missingOps: Uint8Array = doc.exportFrom(peerFrontier);

// Import ops received from another peer
doc.import(receivedOps);

// Check if this doc has all ops up to a given frontier
const isUpToDate: boolean = doc.isAncestor(frontier, doc.version());
```

---

## 4. ProseMirror ↔ Loro Binding

The binding is a bidirectional bridge between ProseMirror's transaction model and Loro's operation model. It lives in `packages/crdt/src/pm-loro-binding.ts`.

### Architecture

```
ProseMirror Editor
    │   ▲
    │   │ applyRemoteTransaction()
    ▼   │
LoroBinding (the bridge)
    │   ▲
    │   │ onLoroUpdate()
    ▼   │
LoroDoc (CRDT state)
    │   ▲
    │   │
    ▼   │
WebSocket Gateway (sync)
```

### PM Schema Mapping

The ProseMirror schema is a flat representation of the hierarchical AST. Structural hierarchy (acts, scenes) is represented in the LoroTree, not in ProseMirror. ProseMirror only sees the **content** within a scene — the scene heading, action blocks, dialogue groups — as a flat document.

```typescript
// ProseMirror schema for screenplay elements
import { Schema } from 'prosemirror-model';

export const screenplaySchema = new Schema({
  nodes: {
    doc: { content: 'block+' },

    scene_heading: {
      content: 'text*',
      marks: '',
      group: 'block',
      attrs: { nodeId: { default: null } },
      parseDOM: [{ tag: 'p[data-type="scene_heading"]' }],
      toDOM: (node) => ['p', { 'data-type': 'scene_heading', 'data-node-id': node.attrs.nodeId }, 0],
    },

    action: {
      content: 'text*',
      marks: 'em strong underline',
      group: 'block',
      attrs: { nodeId: { default: null }, centered: { default: false } },
      parseDOM: [{ tag: 'p[data-type="action"]' }],
      toDOM: (node) => ['p', { 'data-type': 'action', 'data-node-id': node.attrs.nodeId }, 0],
    },

    dialogue_group: {
      content: 'character_name parenthetical? dialogue+',
      group: 'block',
      attrs: { nodeId: { default: null }, character_ref: { default: null } },
      toDOM: (node) => ['div', { 'data-type': 'dialogue_group', 'data-node-id': node.attrs.nodeId }, 0],
    },

    character_name: {
      content: 'text*',
      marks: '',
      attrs: { nodeId: { default: null }, extension: { default: null } },
      toDOM: (node) => ['p', { 'data-type': 'character_name', 'data-node-id': node.attrs.nodeId }, 0],
    },

    parenthetical: {
      content: 'text*',
      marks: '',
      attrs: { nodeId: { default: null } },
      toDOM: (node) => ['p', { 'data-type': 'parenthetical', 'data-node-id': node.attrs.nodeId }, 0],
    },

    dialogue: {
      content: 'text*',
      marks: 'em strong underline',
      attrs: { nodeId: { default: null } },
      toDOM: (node) => ['p', { 'data-type': 'dialogue', 'data-node-id': node.attrs.nodeId }, 0],
    },

    transition: {
      content: 'text*',
      marks: '',
      group: 'block',
      attrs: { nodeId: { default: null }, transition_type: { default: 'custom' } },
      toDOM: (node) => ['p', { 'data-type': 'transition', 'data-node-id': node.attrs.nodeId }, 0],
    },

    text: { inline: true, group: 'inline' },
  },

  marks: {
    em:        { parseDOM: [{ tag: 'em' }], toDOM: () => ['em', 0] },
    strong:    { parseDOM: [{ tag: 'strong' }], toDOM: () => ['strong', 0] },
    underline: { parseDOM: [{ tag: 'u' }], toDOM: () => ['u', 0] },
  },
});
```

### Binding Class

```typescript
import { EditorState, Transaction } from 'prosemirror-state';
import { EditorView } from 'prosemirror-view';
import { LoroDoc, LoroText, Delta } from 'loro-crdt';

export class LoroPMBinding {
  private view: EditorView;
  private doc: LoroDoc;
  private _isApplyingRemote = false;
  private _unsubscribe: (() => void) | null = null;

  constructor(view: EditorView, doc: LoroDoc) {
    this.view = view;
    this.doc = doc;
    this._subscribeToLoro();
  }

  // ── ProseMirror → Loro ───────────────────────────────────────────────────

  /**
   * Called by the ProseMirror plugin on every local transaction.
   * Skip if we're currently applying a remote update to avoid loops.
   */
  onPMTransaction(tr: Transaction): void {
    if (this._isApplyingRemote) return;
    if (!tr.docChanged) return;

    // Map each step to Loro operations
    tr.steps.forEach((step, i) => {
      const stepType = step.constructor.name;

      if (stepType === 'ReplaceStep') {
        this._applyReplaceStep(step as any, tr.docs[i]);
      } else if (stepType === 'ReplaceAroundStep') {
        this._applyReplaceAroundStep(step as any, tr.docs[i]);
      }
      // AddMarkStep and RemoveMarkStep handled separately (marks → Loro text annotations)
    });
  }

  private _applyReplaceStep(step: any, docBefore: any): void {
    const { from, to, slice } = step;

    // Find which PM node(s) are affected by this range
    const affectedNodes = this._getAffectedLeafNodes(docBefore, from, to);

    for (const { nodeId, localFrom, localTo } of affectedNodes) {
      const loroText = getNodeText(this.doc, nodeId);

      // Delete the replaced range
      if (localTo > localFrom) {
        loroText.delete(localFrom, localTo - localFrom);
      }

      // Insert the new content
      if (slice.content.size > 0) {
        const insertedText = slice.content.textBetween(0, slice.content.size);
        if (insertedText) {
          loroText.insert(localFrom, insertedText);
        }
      }
    }
  }

  /**
   * Map a ProseMirror document position to a {nodeId, localOffset} pair.
   * Every leaf PM node has a data-node-id attribute linking it to an AST UUID.
   */
  private _getAffectedLeafNodes(
    doc: any,
    from: number,
    to: number,
  ): Array<{ nodeId: string; localFrom: number; localTo: number }> {
    const result: Array<{ nodeId: string; localFrom: number; localTo: number }> = [];

    doc.nodesBetween(from, to, (node: any, pos: number) => {
      if (!node.isLeaf && node.attrs.nodeId) {
        const nodeStart = pos + 1; // +1 to skip the node's opening token
        const nodeEnd = pos + node.nodeSize - 1;
        const localFrom = Math.max(0, from - nodeStart);
        const localTo = Math.min(node.content.size, to - nodeStart);
        result.push({ nodeId: node.attrs.nodeId, localFrom, localTo });
      }
    });

    return result;
  }

  // ── Loro → ProseMirror ───────────────────────────────────────────────────

  private _subscribeToLoro(): void {
    // Subscribe to text changes across all LoroText containers in this doc
    const unsubscribe = this.doc.subscribe((event) => {
      if (event.local) return; // our own changes — already reflected in PM
      this._applyLoroUpdate(event);
    });
    this._unsubscribe = unsubscribe;
  }

  private _applyLoroUpdate(event: any): void {
    this._isApplyingRemote = true;
    try {
      const { view } = this;
      let tr = view.state.tr;

      for (const [containerId, delta] of Object.entries(event.events)) {
        if (!containerId.startsWith('text:')) continue;
        const nodeId = containerId.slice(5); // strip "text:" prefix

        // Find the PM node with this nodeId
        const pmPos = this._findPMNodePosition(view.state.doc, nodeId);
        if (pmPos === null) continue;

        // Convert Loro Delta to ProseMirror steps
        tr = this._applyDeltaToPM(tr, view.state.doc, pmPos, delta as Delta[]);
      }

      if (tr.docChanged) {
        // Mark transaction as remote so onPMTransaction ignores it
        tr.setMeta('remote', true);
        view.dispatch(tr);
      }
    } finally {
      this._isApplyingRemote = false;
    }
  }

  private _applyDeltaToPM(
    tr: Transaction,
    doc: any,
    nodeStart: number,
    delta: Delta[],
  ): Transaction {
    let offset = 0;

    for (const op of delta) {
      if ('retain' in op) {
        offset += op.retain as number;
      } else if ('insert' in op) {
        const text = op.insert as string;
        tr = tr.insertText(text, nodeStart + offset);
        offset += text.length;
      } else if ('delete' in op) {
        const count = op.delete as number;
        tr = tr.delete(nodeStart + offset, nodeStart + offset + count);
        // Note: don't advance offset — the text was deleted
      }
    }

    return tr;
  }

  private _findPMNodePosition(doc: any, nodeId: string): number | null {
    let result: number | null = null;
    doc.descendants((node: any, pos: number) => {
      if (node.attrs?.nodeId === nodeId) {
        result = pos + 1; // +1 = start of node content
        return false; // stop traversal
      }
    });
    return result;
  }

  destroy(): void {
    this._unsubscribe?.();
  }
}
```

### ProseMirror Plugin

The plugin integrates the binding into ProseMirror's transaction lifecycle:

```typescript
import { Plugin, PluginKey } from 'prosemirror-state';

export const loroPluginKey = new PluginKey('loro');

export function loroPlugin(doc: LoroDoc): Plugin {
  return new Plugin({
    key: loroPluginKey,

    view(editorView) {
      const binding = new LoroPMBinding(editorView, doc);
      return {
        update(view, prevState) {
          // Dispatch local transactions to Loro
          const lastTr = view.state.tr; // access via view.state.storedMarks trick
          // (actual implementation hooks into the dispatch chain)
        },
        destroy() {
          binding.destroy();
        },
      };
    },

    props: {
      handleDOMEvents: {
        // Intercept composition events (IME — Asian character input)
        compositionend: (view, event) => {
          // Force a CRDT sync on IME commit
          return false;
        },
      },
    },
  });
}
```

### Mark Handling

ProseMirror marks (bold, italic, underline) map to Loro's rich text annotation system:

```typescript
// Applying a mark to a range
function applyMark(
  loroText: LoroText,
  from: number,
  to: number,
  markType: 'bold' | 'italic' | 'underline',
  value: boolean,
): void {
  loroText.mark({ start: from, end: to }, markType, value);
}

// Reading marks back (for PM → Loro on mark steps)
function syncMarksToLoro(
  loroText: LoroText,
  from: number,
  to: number,
  marks: readonly any[],
): void {
  // Remove all marks in range first, then re-apply
  loroText.mark({ start: from, end: to }, 'bold', false);
  loroText.mark({ start: from, end: to }, 'italic', false);
  loroText.mark({ start: from, end: to }, 'underline', false);

  for (const mark of marks) {
    loroText.mark({ start: from, end: to }, mark.type.name, true);
  }
}
```

---

## 5. WebSocket Sync Protocol

The Collaboration Gateway uses a binary-framed WebSocket protocol. Messages are encoded as a 1-byte type tag followed by the payload.

### Message Type Registry

```typescript
export const enum MsgType {
  // Handshake (sync)
  SyncStep1   = 0x01,  // Client → Server: "here is my frontier"
  SyncStep2   = 0x02,  // Server → Client: "here are ops you're missing"
  SyncDone    = 0x03,  // Server → Client: "you are now up to date"

  // Live ops
  Update      = 0x10,  // Both directions: incremental CRDT ops

  // Structural ops (require coordination lock)
  StructOp    = 0x20,  // Client → Server: structural operation request
  StructAck   = 0x21,  // Server → Client: structural op applied
  StructNack  = 0x22,  // Server → Client: structural op rejected (lock contention)

  // Awareness
  Awareness   = 0x30,  // Both directions: cursor/presence state

  // Control
  Error       = 0xE0,
  Ping        = 0xF0,
  Pong        = 0xF1,
}
```

### Message Encoding

```typescript
// All messages: [1 byte type][payload bytes]

// SyncStep1: [0x01][frontier: Uint8Array]
function encodeSyncStep1(frontier: Uint8Array): Uint8Array {
  return concat([new Uint8Array([MsgType.SyncStep1]), frontier]);
}

// SyncStep2: [0x02][update: Uint8Array]
function encodeSyncStep2(update: Uint8Array): Uint8Array {
  return concat([new Uint8Array([MsgType.SyncStep2]), update]);
}

// Update: [0x10][4-byte origin length][origin string][update: Uint8Array]
function encodeUpdate(update: Uint8Array, origin: string): Uint8Array {
  const originBytes = new TextEncoder().encode(origin);
  const lenBytes = new Uint8Array(4);
  new DataView(lenBytes.buffer).setUint32(0, originBytes.length, false);
  return concat([new Uint8Array([MsgType.Update]), lenBytes, originBytes, update]);
}

// StructOp: [0x20][JSON payload as UTF-8]
function encodeStructOp(op: StructuralOp): Uint8Array {
  const json = new TextEncoder().encode(JSON.stringify(op));
  return concat([new Uint8Array([MsgType.StructOp]), json]);
}
```

### Handshake Sequence

```
Client connects via WebSocket upgrade:
  GET /collab/edit/{script_id}
  Authorization: Bearer {jwt}
  Sec-WebSocket-Protocol: scriptos-crdt-v1

Server validates JWT → injects user context → accepts connection

Client → Server:  SyncStep1 { frontier: client's current version vector }
                  (if client is new: empty frontier)

Server:
  1. Load the script's current Loro state from Redis (or Postgres crdt_snapshot)
  2. Compute missing = doc.exportFrom(clientFrontier)
  3. Reply: SyncStep2 { update: missing ops }
  4. Reply: SyncDone {}

Client applies SyncStep2 ops → is now synchronized

Client → Server:  SyncStep1 { frontier: updated frontier after applying SyncStep2 }
                  (acknowledgement — allows server to trim Redis stream)

Live editing begins: both directions send Update messages as ops are generated
```

### Gateway Message Handler (TypeScript, Node.js)

```typescript
import { WebSocket } from 'ws';
import { LoroDoc } from 'loro-crdt';
import { getRedisClient } from '../redis';

export async function handleCollabConnection(
  ws: WebSocket,
  scriptId: string,
  userId: string,
): Promise<void> {
  const redis = getRedisClient();

  // Subscribe to the Redis stream for this script
  const streamKey = `crdt:${scriptId}`;
  const sub = redis.duplicate();

  // Load the current doc state from Redis or Postgres
  const serverDoc = await loadDocState(scriptId);

  ws.on('message', async (data: Buffer) => {
    const msgType = data[0];
    const payload = data.slice(1);

    switch (msgType) {
      case MsgType.SyncStep1: {
        const clientFrontier = payload;
        const missingOps = serverDoc.exportFrom(clientFrontier);

        ws.send(encodeSyncStep2(missingOps));
        ws.send(new Uint8Array([MsgType.SyncDone]));
        break;
      }

      case MsgType.Update: {
        // Apply to server doc
        serverDoc.import(payload);

        // Fan out via Redis Stream (all other WS servers will pick this up)
        await redis.xadd(streamKey, '*',
          'type', 'update',
          'origin', userId,
          'data', Buffer.from(payload).toString('base64'),
        );
        break;
      }

      case MsgType.StructOp: {
        const op: StructuralOp = JSON.parse(new TextDecoder().decode(payload));
        await handleStructuralOp(ws, scriptId, serverDoc, op, userId, redis);
        break;
      }

      case MsgType.Awareness: {
        // Fan out awareness state — do NOT apply to serverDoc (ephemeral)
        await redis.publish(`awareness:${scriptId}`, JSON.stringify({
          userId,
          data: Buffer.from(payload.slice(1)).toString('base64'),
        }));
        break;
      }

      case MsgType.Ping:
        ws.send(new Uint8Array([MsgType.Pong]));
        break;
    }
  });

  ws.on('close', () => {
    sub.quit();
    onClientDisconnect(scriptId, userId);
  });
}
```

---

## 6. Awareness Protocol

Awareness carries ephemeral state — cursor positions, selection ranges, user identity, and online status. It is NOT part of the CRDT document and is never persisted.

### Awareness State Schema

```typescript
interface AwarenessState {
  userId: string;
  displayName: string;
  color: string;          // CSS hex color, deterministically derived from userId
  cursor: CursorPosition | null;
  selection: SelectionRange | null;
  activeNodeId: string | null;  // AST node UUID the cursor is in
  lastSeen: number;             // Unix timestamp ms (for TTL cleanup)
}

interface CursorPosition {
  nodeId: string;    // AST node UUID
  offset: number;    // character offset within that node's text
}

interface SelectionRange {
  anchor: CursorPosition;
  head: CursorPosition;
}
```

### Wire Format

Awareness messages are JSON encoded as the payload of a `MsgType.Awareness` WebSocket frame:

```typescript
interface AwarenessMessage {
  // Protocol version — allows future format changes
  v: 1;

  // The sending user's state (null = disconnected / clearing state)
  state: AwarenessState | null;

  // Optional: piggybacked states from other users the server knows about
  // (sent to new joiners along with SyncDone to bootstrap their presence UI)
  peers?: Record<string, AwarenessState>;
}
```

### Server-Side Awareness Store

```typescript
// Redis hash: presence:{script_id}
// Field: userId
// Value: JSON-encoded AwarenessState
// TTL: 30 seconds (renewed on each awareness message; cleared on disconnect)

async function updateAwareness(
  scriptId: string,
  userId: string,
  state: AwarenessState,
): Promise<void> {
  const key = `presence:${scriptId}`;
  const pipeline = redis.pipeline();
  pipeline.hset(key, userId, JSON.stringify(state));
  pipeline.expire(key, 30);
  await pipeline.exec();

  // Broadcast to all connected clients for this script
  await redis.publish(`awareness:${scriptId}`, JSON.stringify({
    v: 1,
    state,
  }));
}

async function clearAwareness(scriptId: string, userId: string): Promise<void> {
  await redis.hdel(`presence:${scriptId}`, userId);
  await redis.publish(`awareness:${scriptId}`, JSON.stringify({
    v: 1,
    state: null,  // null = user has left
    userId,       // identify which user left
  }));
}

async function getAwareness(
  scriptId: string,
): Promise<Record<string, AwarenessState>> {
  const raw = await redis.hgetall(`presence:${scriptId}`);
  return Object.fromEntries(
    Object.entries(raw).map(([uid, json]) => [uid, JSON.parse(json)])
  );
}
```

### Client-Side Awareness Sender

```typescript
class AwarenessSender {
  private ws: WebSocket;
  private userId: string;
  private _sendTimer: ReturnType<typeof setTimeout> | null = null;

  constructor(ws: WebSocket, userId: string, displayName: string) {
    this.ws = ws;
    this.userId = userId;
  }

  // Debounce: send at most every 100ms during rapid cursor movement
  updateCursor(nodeId: string, offset: number): void {
    if (this._sendTimer) clearTimeout(this._sendTimer);
    this._sendTimer = setTimeout(() => {
      this._send({ cursor: { nodeId, offset }, selection: null, activeNodeId: nodeId });
    }, 100);
  }

  private _send(partial: Partial<AwarenessState>): void {
    const state: AwarenessState = {
      userId: this.userId,
      displayName: this._displayName,
      color: userColor(this.userId),
      lastSeen: Date.now(),
      cursor: null,
      selection: null,
      activeNodeId: null,
      ...partial,
    };
    const payload = new TextEncoder().encode(JSON.stringify({ v: 1, state }));
    const msg = new Uint8Array(1 + payload.length);
    msg[0] = MsgType.Awareness;
    msg.set(payload, 1);
    this.ws.send(msg);
  }
}

// Deterministic color from userId — consistent across sessions
function userColor(userId: string): string {
  const PALETTE = [
    '#E63946', '#2A9D8F', '#E9C46A', '#264653',
    '#F4A261', '#A8DADC', '#457B9D', '#1D3557',
  ];
  let hash = 0;
  for (const c of userId) hash = (hash * 31 + c.charCodeAt(0)) & 0xffffffff;
  return PALETTE[Math.abs(hash) % PALETTE.length];
}
```

---

## 7. Structural Operations — Split, Merge, Move

Structural operations modify the LoroTree. They require a Redis coordination lock because they are multi-step sequences that must not be interleaved.

### Coordination Lock

```typescript
const STRUCT_LOCK_TTL_MS = 10_000; // 10 seconds max hold time

async function acquireStructLock(
  scriptId: string,
  operationId: string,
): Promise<boolean> {
  const key = `struct_lock:${scriptId}`;
  // SET NX PX (set if not exists, expire in ms)
  const result = await redis.set(key, operationId, 'NX', 'PX', STRUCT_LOCK_TTL_MS);
  return result === 'OK';
}

async function releaseStructLock(
  scriptId: string,
  operationId: string,
): Promise<void> {
  // Only release if we hold the lock (Lua script for atomicity)
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  await redis.eval(script, 1, `struct_lock:${scriptId}`, operationId);
}
```

### Structural Op Types

```typescript
type StructuralOp =
  | { kind: 'move_scene';   sceneId: string; targetParentId: string; targetPosition: number }
  | { kind: 'split_scene';  sceneId: string; splitAtElementId: string }
  | { kind: 'merge_scenes'; topSceneId: string; bottomSceneId: string }
  | { kind: 'create_node';  parentId: string; type: ScriptNodeType; position: number; tempId: string }
  | { kind: 'delete_node';  nodeId: string }
  | { kind: 'reorder';      nodeId: string; newPosition: number }
```

### Scene Move

```typescript
async function handleSceneMoveOp(
  doc: LoroDoc,
  op: Extract<StructuralOp, { kind: 'move_scene' }>,
  scriptId: string,
  operationId: string,
): Promise<void> {
  const acquired = await acquireStructLock(scriptId, operationId);
  if (!acquired) throw new StructLockContentionError();

  try {
    const tree = doc.getTree('structure');

    // Find the LoroTree node for this scene
    const sceneTreeNode = findTreeNodeByAstId(tree, op.sceneId);
    const targetParentTreeNode = findTreeNodeByAstId(tree, op.targetParentId);

    if (!sceneTreeNode || !targetParentTreeNode) {
      throw new Error(`Node not found: ${op.sceneId} or ${op.targetParentId}`);
    }

    // Validate: target parent must be an act or sequence (not a scene)
    const targetType = targetParentTreeNode.data.get('type') as string;
    if (targetType !== 'act' && targetType !== 'sequence') {
      throw new InvalidParentError(`Cannot move scene under ${targetType}`);
    }

    // Execute the move — Loro MovableTree handles concurrent move conflicts
    tree.move(sceneTreeNode.id, targetParentTreeNode.id, op.targetPosition);

    // Update the position metadata
    sceneTreeNode.data.set('position', op.targetPosition);

    // Export and broadcast the ops
    const update = doc.exportFrom(/* previous frontier */);
    await broadcastUpdate(scriptId, update);

  } finally {
    await releaseStructLock(scriptId, operationId);
  }
}
```

### Scene Split

Splitting scene X at element E creates a new scene containing elements from E onwards. This is the most complex structural operation.

```typescript
async function handleSceneSplitOp(
  doc: LoroDoc,
  op: Extract<StructuralOp, { kind: 'split_scene' }>,
  scriptId: string,
  operationId: string,
  userId: string,
): Promise<{ newSceneId: string }> {
  const acquired = await acquireStructLock(scriptId, operationId);
  if (!acquired) throw new StructLockContentionError();

  try {
    const tree = doc.getTree('structure');
    const sceneNode = findTreeNodeByAstId(tree, op.sceneId);
    if (!sceneNode) throw new Error(`Scene not found: ${op.sceneId}`);

    const parentNode = sceneNode.parent();
    const scenePosition = sceneNode.data.get('position') as number;

    // Step 1: Create the new scene node immediately after the original
    const newSceneId = generateUUIDv7();
    const newScenePosition = fractionalMidpoint(scenePosition, nextSiblingPosition(tree, sceneNode));
    const newSceneTreeNode = tree.createNode(parentNode!.id);
    newSceneTreeNode.data.set('ast_id', newSceneId);
    newSceneTreeNode.data.set('type', 'scene');
    newSceneTreeNode.data.set('position', newScenePosition);
    newSceneTreeNode.data.set('metadata', JSON.stringify({
      scene_number: null,      // will be assigned on next publish
      scene_number_locked: false,
      omitted: false,
      // other fields start null — inherited from original at checkpoint
    }));

    // Step 2: Create a new scene_heading for the new scene
    // (Copy of original heading — writer can edit after split)
    const origHeadingNode = findFirstChildOfType(tree, sceneNode, 'scene_heading');
    const newHeadingId = generateUUIDv7();
    const newHeadingNode = tree.createNode(newSceneTreeNode.id);
    newHeadingNode.data.set('ast_id', newHeadingId);
    newHeadingNode.data.set('type', 'scene_heading');
    newHeadingNode.data.set('position', 1.0);

    // Copy heading text into new LoroText
    if (origHeadingNode) {
      const origText = getNodeText(doc, origHeadingNode.data.get('ast_id') as string);
      const newText = getNodeText(doc, newHeadingId);
      newText.insert(0, origText.toString());
    }

    // Step 3: Find all children of original scene that come AFTER splitAtElementId
    const children = getSortedChildren(tree, sceneNode);
    const splitIndex = children.findIndex(
      c => c.data.get('ast_id') === op.splitAtElementId
    );
    if (splitIndex === -1) throw new Error(`Split element not found: ${op.splitAtElementId}`);

    const toMove = children.slice(splitIndex);

    // Step 4: Move each element to the new scene (preserving order)
    let newPos = 2.0; // 1.0 is the heading
    for (const child of toMove) {
      tree.move(child.id, newSceneTreeNode.id, newPos);
      child.data.set('position', newPos);
      newPos += 1.0;
    }

    // Step 5: Persist to Postgres via checkpoint (async — don't block the WS response)
    // The checkpoint will create the new ast_node rows and update parent/position fields
    scheduleCheckpoint(scriptId, 'structural_split');

    // Export and broadcast
    const update = doc.exportFrom(/* previous frontier */);
    await broadcastUpdate(scriptId, update);

    return { newSceneId };

  } finally {
    await releaseStructLock(scriptId, operationId);
  }
}
```

### Scene Merge

Merging scene B into scene A moves all of B's elements (except its heading) under A and soft-deletes B.

```typescript
async function handleSceneMergeOp(
  doc: LoroDoc,
  op: Extract<StructuralOp, { kind: 'merge_scenes' }>,
  scriptId: string,
  operationId: string,
): Promise<void> {
  const acquired = await acquireStructLock(scriptId, operationId);
  if (!acquired) throw new StructLockContentionError();

  try {
    const tree = doc.getTree('structure');
    const topScene = findTreeNodeByAstId(tree, op.topSceneId);
    const bottomScene = findTreeNodeByAstId(tree, op.bottomSceneId);

    if (!topScene || !bottomScene) throw new Error('Scene not found');

    // Validate adjacency — scenes must be siblings and consecutive
    if (!areSiblings(tree, topScene, bottomScene)) {
      throw new InvalidMergeError('Scenes must be in the same parent to merge');
    }

    // Get the last position in topScene
    const topChildren = getSortedChildren(tree, topScene);
    let nextPos = (topChildren.at(-1)?.data.get('position') as number ?? 0) + 1.0;

    // Move all bottom scene children (except its heading) to top scene
    const bottomChildren = getSortedChildren(tree, bottomScene);
    for (const child of bottomChildren) {
      if (child.data.get('type') === 'scene_heading') continue; // skip heading

      tree.move(child.id, topScene.id, nextPos);
      child.data.set('position', nextPos);
      nextPos += 1.0;
    }

    // Soft-delete the bottom scene node
    // (Set omitted=true in metadata — don't remove from tree, preserve scene number)
    const meta = getNodeMeta(doc, op.bottomSceneId);
    meta.set('omitted', true);
    meta.set('omitted_at', new Date().toISOString());

    scheduleCheckpoint(scriptId, 'structural_merge');

    const update = doc.exportFrom(/* previous frontier */);
    await broadcastUpdate(scriptId, update);

  } finally {
    await releaseStructLock(scriptId, operationId);
  }
}
```

### Fractional Indexing

All `position` values use fractional indexing to allow insertion without renumbering siblings.

```typescript
/**
 * Compute the position value for a new node inserted between two existing positions.
 * Guarantees: result > before AND result < after (if both exist).
 * Uses the same algorithm as Figma/Linear fractional indexing.
 */
function fractionalMidpoint(before: number | null, after: number | null): number {
  if (before === null && after === null) return 1.0;
  if (before === null) return after! / 2;
  if (after === null) return before + 1.0;

  const mid = (before + after) / 2;

  // Guard against float precision collapse (when before and after are very close)
  if (mid <= before || mid >= after) {
    // Rebalance: this branch should be triggered by a background rebalance job,
    // not inline. For now, add a tiny epsilon and log a warning.
    console.warn('Fractional position precision exhausted — rebalance needed', { before, after });
    return before + (after - before) * 0.5 + Number.EPSILON;
  }

  return mid;
}
```

---

## 8. Checkpoint Protocol (CRDT → Postgres)

A checkpoint extracts the current CRDT state into Postgres. The CRDT is always authoritative during active editing; Postgres is authoritative for everything else (exports, breakdown, search, AI grounding, scheduling).

### Checkpoint Triggers

```typescript
type CheckpointTrigger =
  | 'explicit_save'         // User pressed Ctrl+S or Save button
  | 'idle_timeout'          // 5 minutes without editing activity
  | 'session_disconnect'    // Client disconnected (30-second grace)
  | 'publish_preflight'     // Before starting Temporal publish saga (mandatory, blocking)
  | 'structural_split'      // After scene split (create new Postgres rows)
  | 'structural_merge'      // After scene merge (update Postgres rows)
  | 'manual_version_lock'   // User created a named version
  | 'admin_force';          // Admin-triggered (migrations, repairs)
```

### Checkpoint Procedure

```typescript
interface CheckpointInput {
  scriptId: string;
  trigger: CheckpointTrigger;
  doc: LoroDoc;
  previousFrontier: Uint8Array;
}

interface CheckpointResult {
  nodesUpdated: number;
  scenesDerivationUpdated: number;
  newNodesCreated: number;           // for structural split/merge
  versionCreated: string | null;     // only for manual_version_lock
  frontierAfter: Uint8Array;
}

async function checkpoint(input: CheckpointInput): Promise<CheckpointResult> {
  const { scriptId, trigger, doc } = input;

  // 1. Extract all dirty leaf node text content
  const textUpdates: Array<{ nodeId: string; content: string }> = [];
  const tree = doc.getTree('structure');
  const allNodes = getAllTreeNodes(tree);

  for (const treeNode of allNodes) {
    const astId = treeNode.data.get('ast_id') as string;
    const type = treeNode.data.get('type') as ScriptNodeType;

    if (isLeafNodeType(type)) {
      const text = getNodeText(doc, astId);
      textUpdates.push({ nodeId: astId, content: text.toString() });
    }
  }

  // 2. Extract structural changes (new nodes, moved nodes, deleted nodes)
  const structuralChanges = extractStructuralChanges(doc, tree, input.previousFrontier);

  // 3. Write to Postgres in a single transaction
  const result = await db.transaction(async (tx) => {
    let nodesUpdated = 0;
    let newNodesCreated = 0;

    // 3a. Handle new nodes (from structural split/merge)
    for (const newNode of structuralChanges.created) {
      await tx.query(`
        INSERT INTO script.ast_nodes
          (id, script_id, type, parent_id, position, content_snapshot, metadata,
           created_at, created_by, updated_at, updated_by, version)
        VALUES ($1, $2, $3, $4, $5, $6, $7, NOW(), $8, NOW(), $8, 1)
        ON CONFLICT (id) DO NOTHING
      `, [
        newNode.astId, scriptId, newNode.type, newNode.parentId,
        newNode.position, newNode.contentSnapshot, JSON.stringify(newNode.metadata),
        newNode.createdBy,
      ]);
      newNodesCreated++;
    }

    // 3b. Update moved nodes (parent_id and position)
    for (const moved of structuralChanges.moved) {
      await tx.query(`
        UPDATE script.ast_nodes
        SET parent_id = $2, position = $3, updated_at = NOW(), version = version + 1
        WHERE id = $1
      `, [moved.astId, moved.newParentId, moved.newPosition]);
    }

    // 3c. Update soft-deleted nodes
    for (const deleted of structuralChanges.deleted) {
      await tx.query(`
        UPDATE script.ast_nodes
        SET deleted_at = NOW(), metadata = metadata || '{"omitted": true}'::jsonb,
            version = version + 1
        WHERE id = $1
      `, [deleted.astId]);
    }

    // 3d. Update text content for all dirty leaf nodes
    for (const { nodeId, content } of textUpdates) {
      const res = await tx.query(`
        UPDATE script.ast_nodes
        SET content_snapshot = $2, updated_at = NOW(), version = version + 1
        WHERE id = $1 AND content_snapshot IS DISTINCT FROM $2
        RETURNING id
      `, [nodeId, content]);
      if (res.rowCount > 0) nodesUpdated++;
    }

    // 3e. Re-derive scene metadata for all scenes with dirty children
    const dirtySceneIds = getDirtySceneIds(structuralChanges, textUpdates, allNodes);
    let scenesDerivationUpdated = 0;

    for (const sceneId of dirtySceneIds) {
      const derived = deriveSceneMetadata(doc, tree, sceneId);
      await tx.query(`
        UPDATE script.ast_nodes
        SET metadata = metadata || $2::jsonb, updated_at = NOW(), version = version + 1
        WHERE id = $1 AND type = 'scene'
      `, [sceneId, JSON.stringify(derived)]);
      scenesDerivationUpdated++;
    }

    // 3f. For manual_version_lock: create script_versions snapshot
    let versionCreated: string | null = null;
    if (trigger === 'manual_version_lock' || trigger === 'publish_preflight') {
      const snapshot = await buildVersionSnapshot(tx, scriptId);
      const crdtSnapshot = doc.exportSnapshot(); // full Loro state as bytes
      const versionId = generateUUIDv7();

      await tx.query(`
        INSERT INTO script.script_versions
          (id, script_id, version_number, revision_color, snapshot, crdt_snapshot,
           page_count, locked, created_at, created_by)
        SELECT $1, $2, COALESCE(MAX(version_number), 0) + 1,
               s.revision_color, $3, $4, $5, $6, NOW(), $7
        FROM script.scripts s
        WHERE s.id = $2
        GROUP BY s.revision_color
      `, [versionId, scriptId, JSON.stringify(snapshot),
          Buffer.from(crdtSnapshot), snapshot.pageCount,
          trigger === 'publish_preflight', snapshot.createdBy]);

      versionCreated = versionId;
    }

    return { nodesUpdated, scenesDerivationUpdated, newNodesCreated, versionCreated };
  });

  // 4. Update the Redis stream trim marker (operations before this frontier are safe to trim)
  const frontierAfter = doc.version();
  await redis.set(`crdt:frontier:${scriptId}`, Buffer.from(frontierAfter).toString('base64'));

  // 5. Publish checkpoint event to NATS
  await nats.publish('script.checkpointed', {
    script_id: scriptId,
    dirty_node_ids: textUpdates.map(u => u.nodeId),
    dirty_scene_ids: [...getDirtySceneIds(structuralChanges, textUpdates, allNodes)],
    checkpoint_type: trigger === 'publish_preflight' ? 'publish' :
                     trigger === 'explicit_save'      ? 'manual' : 'auto',
  });

  return { ...result, frontierAfter };
}
```

### Scene Metadata Derivation

Called during checkpoint for scenes with dirty children — re-derives cached metadata from CRDT content:

```typescript
interface DerivedSceneMetadata {
  int_ext: string | null;
  location_name: string | null;
  time_of_day: string | null;
  speaking_characters: string[];  // node UUIDs of character_name children
}

function deriveSceneMetadata(
  doc: LoroDoc,
  tree: LoroTree,
  sceneId: string,
): DerivedSceneMetadata {
  const sceneNode = findTreeNodeByAstId(tree, sceneId);
  if (!sceneNode) return { int_ext: null, location_name: null, time_of_day: null, speaking_characters: [] };

  const children = getSortedChildren(tree, sceneNode);
  let heading: string | null = null;
  const speakingCharacters: string[] = [];

  for (const child of children) {
    const type = child.data.get('type') as string;
    const astId = child.data.get('ast_id') as string;

    if (type === 'scene_heading') {
      heading = getNodeText(doc, astId).toString();
    } else if (type === 'dialogue_group') {
      // Find the character_name child
      const charChildren = getSortedChildren(tree, child);
      for (const c of charChildren) {
        if (c.data.get('type') === 'character_name') {
          speakingCharacters.push(c.data.get('ast_id') as string);
        }
      }
    }
  }

  return {
    ...parseSceneHeading(heading),
    speaking_characters: speakingCharacters,
  };
}

function parseSceneHeading(heading: string | null): {
  int_ext: string | null;
  location_name: string | null;
  time_of_day: string | null;
} {
  if (!heading) return { int_ext: null, location_name: null, time_of_day: null };

  // Pattern: "INT. LOCATION - TIME" or "EXT./INT. LOCATION (CONT'D) - TIME"
  const match = heading.match(
    /^(INT\.|EXT\.|INT\.\/EXT\.|EXT\.\/INT\.|I\/E)\s+(.+?)(?:\s+-\s+(.+))?$/i
  );

  if (!match) return { int_ext: null, location_name: heading, time_of_day: null };

  return {
    int_ext: match[1].toUpperCase(),
    location_name: match[2].trim(),
    time_of_day: match[3]?.trim() ?? null,
  };
}
```

---

## 9. Compaction Algorithm

CRDT documents grow over time as operations accumulate. Compaction creates a clean snapshot and trims history.

### When to Compact

```typescript
type CompactionTrigger =
  | 'revision_color_change'  // Mandatory: a new revision color is formally issued
  | 'daily_high_activity'    // Scripts with > 1,000 ops since last compaction
  | 'admin_requested';       // Manual trigger for specific scripts

// Evaluated by a daily background job in the Script Service
async function shouldCompact(scriptId: string): Promise<CompactionTrigger | null> {
  const [opCount, lastCompaction, lastRevision] = await Promise.all([
    redis.xlen(`crdt:${scriptId}`),
    redis.get(`crdt:last_compaction:${scriptId}`),
    db.query(`SELECT revision_color, updated_at FROM script.scripts WHERE id = $1`, [scriptId]),
  ]);

  const lastCompactionTime = lastCompaction ? parseInt(lastCompaction) : 0;
  const lastRevisionTime = new Date(lastRevision.rows[0]?.updated_at).getTime();

  if (lastRevisionTime > lastCompactionTime) {
    return 'revision_color_change';
  }

  if (opCount > 1_000) {
    return 'daily_high_activity';
  }

  return null;
}
```

### Compaction Procedure

```typescript
async function compactDocument(
  scriptId: string,
  trigger: CompactionTrigger,
): Promise<void> {
  // 1. Load the current doc state (from Redis stream + Postgres snapshot base)
  const doc = await loadDocState(scriptId);

  // 2. Create a clean snapshot (full document state, no incremental history)
  const snapshot: Uint8Array = doc.exportSnapshot();
  // exportSnapshot() in Loro creates a minimal representation — all ops are
  // collapsed into the current state. The resulting bytes are much smaller
  // than the full op log (typically 3–10x compression).

  // 3. Write the snapshot to S3 for archive (in case of future forensic need)
  const s3Key = `snapshots/${scriptId}/${Date.now()}.loro`;
  await s3.putObject({ Key: s3Key, Body: Buffer.from(snapshot) });

  // 4. Update the crdt_snapshot in script_versions (most recent version)
  await db.query(`
    UPDATE script.script_versions
    SET crdt_snapshot = $2
    WHERE script_id = $1
      AND version_number = (
        SELECT MAX(version_number) FROM script.script_versions WHERE script_id = $1
      )
  `, [scriptId, Buffer.from(snapshot)]);

  // 5. Replace the Redis Stream with the new snapshot as the baseline
  // (Trim the stream to remove all ops; future ops build on the snapshot)
  const streamKey = `crdt:${scriptId}`;
  const pipeline = redis.pipeline();

  // Store the snapshot as the bootstrap for new connections
  pipeline.set(`crdt:snapshot:${scriptId}`, Buffer.from(snapshot).toString('base64'));

  // Trim the stream — all ops before this point are now in the snapshot
  pipeline.xtrim(streamKey, 'MAXLEN', 0);

  // Record compaction timestamp
  pipeline.set(`crdt:last_compaction:${scriptId}`, Date.now().toString());

  await pipeline.exec();

  // 6. Notify connected clients: they should reload their doc from the new snapshot
  // (Existing sessions continue working — their in-memory doc is already up to date.
  // New sessions joining after compaction will load from the snapshot.)
  await redis.publish(`collab:notify:${scriptId}`, JSON.stringify({
    type: 'compacted',
    snapshotKey: s3Key,
  }));
}
```

### Loading Doc State (accounting for compaction)

```typescript
async function loadDocState(scriptId: string): Promise<LoroDoc> {
  const doc = new LoroDoc();

  // 1. Check for a compacted snapshot in Redis
  const snapshotBase64 = await redis.get(`crdt:snapshot:${scriptId}`);

  if (snapshotBase64) {
    // Start from the compacted snapshot
    const snapshot = Buffer.from(snapshotBase64, 'base64');
    doc.importSnapshot(snapshot);
  } else {
    // Fall back to Postgres crdt_snapshot (first session, or Redis eviction)
    const row = await db.query(`
      SELECT crdt_snapshot FROM script.script_versions
      WHERE script_id = $1
      ORDER BY version_number DESC
      LIMIT 1
    `, [scriptId]);

    if (row.rows[0]?.crdt_snapshot) {
      doc.importSnapshot(new Uint8Array(row.rows[0].crdt_snapshot));
    } else {
      // No CRDT state yet — bootstrap from Postgres AST nodes
      const nodes = await loadASTNodes(scriptId);
      bootstrapDocFromAST(doc, nodes);
    }
  }

  // 2. Apply any ops in the Redis Stream since the snapshot
  const streamKey = `crdt:${scriptId}`;
  const ops = await redis.xrange(streamKey, '-', '+');

  for (const [, fields] of ops) {
    const dataIdx = fields.indexOf('data');
    if (dataIdx >= 0) {
      const opBytes = Buffer.from(fields[dataIdx + 1], 'base64');
      doc.import(new Uint8Array(opBytes));
    }
  }

  return doc;
}
```

---

## 10. Sync Infrastructure — y-redis / Hocuspocus

### Startup Phase: Hocuspocus (MIT, single instance)

At startup scale (< 1,000 active users, < 20 concurrent sessions per document), Hocuspocus runs as a single process. It stores document state in-memory and persists to Postgres via the Hocuspocus Postgres extension.

```typescript
// packages/collab/src/server-hocuspocus.ts
import { Server } from '@hocuspocus/server';
import { Logger } from '@hocuspocus/extension-logger';
import { Database } from '@hocuspocus/extension-database';

// NOTE: Hocuspocus uses Yjs internally. At startup, we run the Yjs fallback (§11).
// We switch to the Loro + y-redis path when horizontal scaling is needed.
// The client-facing WebSocket protocol is the same for both.

export const hocuspocusServer = Server.configure({
  port: 1234,
  extensions: [
    new Logger({ log: logger.info.bind(logger) }),
    new Database({
      fetch: async ({ documentName }) => {
        // Load from Postgres crdt_snapshot (yjsState column)
        const row = await db.query(
          `SELECT yjs_state FROM script.script_yjs_state WHERE script_id = $1`,
          [documentName]
        );
        return row.rows[0]?.yjs_state ?? null;
      },
      store: async ({ documentName, state }) => {
        await db.query(`
          INSERT INTO script.script_yjs_state (script_id, yjs_state, updated_at)
          VALUES ($1, $2, NOW())
          ON CONFLICT (script_id) DO UPDATE SET yjs_state = $2, updated_at = NOW()
        `, [documentName, state]);
      },
    }),
  ],

  async onAuthenticate({ token }) {
    const claims = await validateJWT(token);
    if (!claims) throw new Error('Unauthorized');
    return { userId: claims.sub, orgId: claims.org_id };
  },

  async onConnect({ documentName, context }) {
    // Check project access
    const hasAccess = await checkScriptAccess(context.userId, documentName);
    if (!hasAccess) throw new Error('Forbidden');
  },
});
```

### Growth Phase: y-redis (AGPL, multi-instance)

When horizontal scaling is needed, replace Hocuspocus with y-redis. The client-side code is unchanged — the WebSocket protocol is wire-compatible.

#### Kubernetes Deployment Spec

```yaml
# infra/k8s/collab-gateway.yaml

# WebSocket Server Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collab-gateway
  namespace: scriptos-gateway
spec:
  replicas: 5          # Start at 5; HPA scales to 50 based on connection count
  selector:
    matchLabels:
      app: collab-gateway
  template:
    metadata:
      labels:
        app: collab-gateway
    spec:
      containers:
        - name: collab-gateway
          image: scriptos/collab-gateway:latest
          ports:
            - containerPort: 8080
          env:
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
            - name: AUTH_SERVICE_URL
              value: "auth-service.scriptos-core.svc.cluster.local:9000"
            - name: SCRIPT_SERVICE_URL
              value: "script-service.scriptos-core.svc.cluster.local:9000"
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "2Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5

---
# HPA for WebSocket Gateway
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: collab-gateway-hpa
  namespace: scriptos-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: collab-gateway
  minReplicas: 5
  maxReplicas: 50
  metrics:
    - type: External
      external:
        metric:
          name: websocket_active_connections
          selector:
            matchLabels:
              app: collab-gateway
        target:
          type: AverageValue
          averageValue: "5000"  # scale up when any pod exceeds 5,000 connections

---
# y-redis Worker Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crdt-worker
  namespace: scriptos-core
spec:
  replicas: 2
  selector:
    matchLabels:
      app: crdt-worker
  template:
    spec:
      containers:
        - name: crdt-worker
          image: scriptos/crdt-worker:latest
          env:
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: redis-credentials
                  key: url
            - name: POSTGRES_URL
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: url
            - name: CHECKPOINT_IDLE_SECONDS
              value: "300"    # 5-minute idle checkpoint
            - name: MAX_OPS_BEFORE_CHECKPOINT
              value: "10000"  # also checkpoint if ops accumulate
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "4Gi"

---
# Redis Streams Configuration (applied via ConfigMap + init job)
# Stream: crdt:{script_id}
# Max length: 100,000 entries (auto-trimmed by MAXLEN ~)
# TTL: 30 days after last write (set by worker on each trim)
```

#### Worker Checkpoint Loop

```typescript
// packages/crdt/src/worker.ts
// The y-redis worker — runs as a separate deployment, not part of the WS server

import { createClient } from 'redis';

async function runWorker(): Promise<void> {
  const redis = createClient({ url: process.env.REDIS_URL });
  await redis.connect();

  // Process all streams with pending updates
  while (true) {
    const scripts = await getActiveScripts(redis);

    for (const scriptId of scripts) {
      const opCount = await redis.xLen(`crdt:${scriptId}`);
      const lastActivity = await redis.get(`crdt:last_activity:${scriptId}`);
      const idleSecs = lastActivity
        ? (Date.now() - parseInt(lastActivity)) / 1000
        : Infinity;

      const shouldCheckpoint =
        idleSecs > CHECKPOINT_IDLE_SECONDS ||
        opCount > MAX_OPS_BEFORE_CHECKPOINT;

      if (shouldCheckpoint) {
        await checkpoint({
          scriptId,
          trigger: 'idle_timeout',
          doc: await loadDocState(scriptId),
          previousFrontier: new Uint8Array(0),
        });
      }
    }

    await sleep(5_000); // poll every 5 seconds
  }
}
```

---

## 11. Yjs Fallback Specification

If the Loro POC fails, this is the complete Yjs fallback. The system design and service boundaries are unchanged — only the CRDT library and binding change.

### Document Model (Yjs)

```typescript
import * as Y from 'yjs';

// One Y.Doc per script (same as Loro: one doc per script UUID)
function createYjsDoc(scriptId: string): Y.Doc {
  const doc = new Y.Doc({ guid: scriptId });
  return doc;
}

// Structure: Y.Map keyed by node UUID, values are Y.Map of node properties
// (Yjs has no native tree move — structure is coordinated, not CRDT)
function getStructureMap(doc: Y.Doc): Y.Map<Y.Map<any>> {
  return doc.getMap('structure');
}

// Text: Y.Text per leaf node, keyed by node UUID
function getNodeText(doc: Y.Doc, nodeId: string): Y.Text {
  return doc.getText(`text:${nodeId}`);
}
```

### Structural Op Coordination (Yjs path)

Under Yjs, structural ops (scene moves, splits, merges) are NOT CRDT. They are serialized via the Redis coordination lock and applied as Y.Map mutations. The lock ensures ordering; Yjs convergence handles the Y.Map updates.

The coordination mechanism is identical to §7 — only the tree mutation API changes (Y.Map.set vs LoroTree.move).

### ProseMirror Binding (Yjs path)

Use `y-prosemirror` directly — no in-house binding needed:

```typescript
import { ySyncPlugin, yUndoPlugin, yCursorPlugin } from 'y-prosemirror';
import { WebsocketProvider } from 'y-websocket';

const ydoc = createYjsDoc(scriptId);
const provider = new WebsocketProvider(
  `wss://collab.scriptos.internal`,
  scriptId,
  ydoc,
  { params: { token: jwt } }
);

const plugins = [
  ySyncPlugin(ydoc.getText(`text:${sceneHeadingId}`)),
  yUndoPlugin(),
  yCursorPlugin(provider.awareness),
];
```

### Sync Infrastructure (Yjs path)

Use y-redis with Yjs (its intended target). The y-redis package handles Redis Stream fan-out for Yjs updates natively.

### Decision Gate at Week 6

| If Loro POC result... | Action |
|-----------------------|--------|
| All 5 pass criteria met | Proceed with Loro path. Yjs fallback is mothballed (kept but not developed). |
| Binding performance fails only | Optimize the binding. 2-week extension. |
| Binding correctness fails | Pivot to Yjs fallback immediately. |
| MovableTree API is unstable | Pivot to Yjs fallback + coordinated structural ops. |

---

## 12. Semantic Validation Layer

CRDTs guarantee convergence, not semantic validity. The validator runs asynchronously after editing settles — it never blocks typing.

### Validation Rules

```typescript
interface ValidationResult {
  valid: boolean;
  errors: ValidationError[];    // blocks publish
  warnings: ValidationError[];  // shown in Problems panel, does not block
}

interface ValidationError {
  nodeId: string;
  rule: string;
  message: string;
  severity: 'error' | 'warning';
}

const VALIDATION_RULES: Array<(doc: LoroDoc, tree: LoroTree) => ValidationError[]> = [

  // Rule 1: Every scene must have exactly one scene_heading as its first child
  (doc, tree) => {
    const errors: ValidationError[] = [];
    for (const node of getNodesByType(tree, 'scene')) {
      const children = getSortedChildren(tree, node);
      const headings = children.filter(c => c.data.get('type') === 'scene_heading');
      if (headings.length === 0) {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'scene_requires_heading',
          message: 'Scene is missing a scene heading (slug line)',
          severity: 'error',
        });
      } else if (children[0].data.get('type') !== 'scene_heading') {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'heading_must_be_first',
          message: 'Scene heading must be the first element in a scene',
          severity: 'error',
        });
      }
    }
    return errors;
  },

  // Rule 2: Every dialogue_group must have exactly one character_name
  (doc, tree) => {
    const errors: ValidationError[] = [];
    for (const node of getNodesByType(tree, 'dialogue_group')) {
      const children = getSortedChildren(tree, node);
      const chars = children.filter(c => c.data.get('type') === 'character_name');
      if (chars.length === 0) {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'dialogue_requires_character',
          message: 'Dialogue block is missing a character name',
          severity: 'error',
        });
      } else if (chars.length > 1) {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'dialogue_single_character',
          message: 'Dialogue block has multiple character names',
          severity: 'error',
        });
      }
    }
    return errors;
  },

  // Rule 3: dual_dialogue must have exactly two dialogue_group children
  (doc, tree) => {
    const errors: ValidationError[] = [];
    for (const node of getNodesByType(tree, 'dual_dialogue')) {
      const children = getSortedChildren(tree, node)
        .filter(c => c.data.get('type') === 'dialogue_group');
      if (children.length !== 2) {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'dual_dialogue_structure',
          message: `Dual dialogue must have exactly 2 dialogue groups, found ${children.length}`,
          severity: 'error',
        });
      }
    }
    return errors;
  },

  // Rule 4: Parenthetical must be inside a dialogue_group
  (doc, tree) => {
    const errors: ValidationError[] = [];
    for (const node of getNodesByType(tree, 'parenthetical')) {
      const parent = node.parent();
      if (!parent || parent.data.get('type') !== 'dialogue_group') {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'orphan_parenthetical',
          message: 'Parenthetical is not inside a dialogue group',
          severity: 'error',
        });
      }
    }
    return errors;
  },

  // Rule 5: Scene heading text must not be empty
  (doc, tree) => {
    const errors: ValidationError[] = [];
    for (const node of getNodesByType(tree, 'scene_heading')) {
      const text = getNodeText(doc, node.data.get('ast_id') as string).toString().trim();
      if (text.length === 0) {
        errors.push({
          nodeId: node.data.get('ast_id') as string,
          rule: 'empty_scene_heading',
          message: 'Scene heading is empty',
          severity: 'error',
        });
      }
    }
    return errors;
  },

  // Warning: Character name not resolved to a known character in the Bible
  (doc, tree) => {
    // This rule runs async (requires Bible lookup) — handled separately
    return [];
  },
];
```

### Validation Runner

```typescript
class ValidationRunner {
  private _timer: ReturnType<typeof setTimeout> | null = null;
  private _DEBOUNCE_MS = 2_000;  // run 2s after last edit settles

  schedule(doc: LoroDoc, tree: LoroTree, onResult: (r: ValidationResult) => void): void {
    if (this._timer) clearTimeout(this._timer);
    this._timer = setTimeout(async () => {
      const result = await validate(doc, tree);
      onResult(result);
    }, this._DEBOUNCE_MS);
  }
}

async function validate(doc: LoroDoc, tree: LoroTree): Promise<ValidationResult> {
  const allErrors: ValidationError[] = [];

  for (const rule of VALIDATION_RULES) {
    allErrors.push(...rule(doc, tree));
  }

  return {
    valid: !allErrors.some(e => e.severity === 'error'),
    errors: allErrors.filter(e => e.severity === 'error'),
    warnings: allErrors.filter(e => e.severity === 'warning'),
  };
}
```

---

## 13. Offline Operation (Tauri On-Set)

The Tauri on-set app uses CRDT for rich text collaboration within the Script Supervisor module (continuity notes, deviation descriptions). These are separate from the main screenplay CRDT — Script Supervisors do not edit the script itself on set.

### Two CRDT Contexts on Set

```
On-Set Tauri App
├── Script content:     READ-ONLY from SQLite (synced via PowerSync from Postgres)
│                       Never edited on set — editorial workflow is separate
└── Supervisor notes:  CRDT text (Loro/Yjs) — edited on set, synced when online
    ├── continuity_notes per scene: LoroText (per-scene continuity observations)
    ├── deviation_description per take: LoroText (per-take deviation description)
    └── supervisor_report sections: LoroText (daily wrap report sections)
```

### Loro State Persistence on Device

```rust
// src-tauri/src/crdt.rs
use loro::LoroDoc;
use rusqlite::Connection;

pub fn save_loro_state(conn: &Connection, doc_id: &str, doc: &LoroDoc) -> Result<()> {
    let snapshot = doc.export_snapshot();
    conn.execute(
        "INSERT INTO crdt_state (doc_id, snapshot, updated_at)
         VALUES (?1, ?2, CURRENT_TIMESTAMP)
         ON CONFLICT(doc_id) DO UPDATE SET snapshot = ?2, updated_at = CURRENT_TIMESTAMP",
        rusqlite::params![doc_id, snapshot],
    )?;
    Ok(())
}

pub fn load_loro_state(conn: &Connection, doc_id: &str) -> Result<LoroDoc> {
    let doc = LoroDoc::new();
    let result: Option<Vec<u8>> = conn
        .query_row(
            "SELECT snapshot FROM crdt_state WHERE doc_id = ?1",
            rusqlite::params![doc_id],
            |row| row.get(0),
        )
        .optional()?;

    if let Some(snapshot) = result {
        doc.import_snapshot(&snapshot)?;
    }

    Ok(doc)
}
```

### Sync on Reconnect

When the on-set device reconnects to the internet:

1. PowerSync syncs structured data (take logs, deviations) to Postgres — handled automatically
2. For CRDT rich text (continuity notes), the Tauri app sends its Loro updates to the Collaboration Service via WebSocket using the standard sync protocol (§5)
3. The Collaboration Service merges the offline ops with any server-side changes (other devices that were on the same local WiFi and synced earlier)
4. Convergence is guaranteed by CRDT properties — no manual conflict resolution needed for text

---

## 14. Error Handling and Recovery

### Connection Errors

```typescript
class CollabClient {
  private _ws: WebSocket | null = null;
  private _reconnectDelay = 1_000;  // start at 1s
  private _MAX_RECONNECT_DELAY = 30_000;  // cap at 30s
  private _reconnectTimer: ReturnType<typeof setTimeout> | null = null;

  connect(scriptId: string, token: string): void {
    this._ws = new WebSocket(
      `wss://collab.scriptos.internal/edit/${scriptId}`,
      'scriptos-crdt-v1'
    );
    this._ws.binaryType = 'arraybuffer';

    this._ws.onopen = () => {
      this._reconnectDelay = 1_000;  // reset backoff on successful connect
      this._performHandshake();
    };

    this._ws.onclose = (event) => {
      if (event.code === 4001) {
        // Auth failure — do not reconnect, show login prompt
        this._onAuthFailure();
        return;
      }
      if (event.code === 4003) {
        // Script locked — do not reconnect, show lock notice
        this._onScriptLocked();
        return;
      }
      // All other closures: exponential backoff reconnect
      this._scheduleReconnect(scriptId, token);
    };

    this._ws.onerror = () => {
      // onerror always precedes onclose — handled there
    };
  }

  private _scheduleReconnect(scriptId: string, token: string): void {
    if (this._reconnectTimer) clearTimeout(this._reconnectTimer);
    this._reconnectTimer = setTimeout(() => {
      this._reconnectDelay = Math.min(
        this._reconnectDelay * 2,
        this._MAX_RECONNECT_DELAY
      );
      this.connect(scriptId, token);
    }, this._reconnectDelay + Math.random() * 1_000);  // jitter
  }
}
```

### Struct Lock Contention

```typescript
// Client-side handling of StructNack (structural op rejected)
function handleStructNack(op: StructuralOp, retryCount: number): void {
  if (retryCount >= 3) {
    // Show user error: "This operation could not be completed. Please try again."
    showToast({ message: 'Scene could not be moved. Another edit is in progress.', type: 'error' });
    return;
  }

  // Retry after a brief delay (the lock holder should release within 10s)
  setTimeout(() => {
    sendStructOp(op, retryCount + 1);
  }, 1_000 * (retryCount + 1));
}
```

### Divergence Detection

```typescript
// Detect if the client's CRDT state has diverged from the server's
// (should be impossible with correct implementation, but caught as a safety net)
async function detectDivergence(
  clientDoc: LoroDoc,
  serverFrontier: Uint8Array,
): Promise<boolean> {
  const clientFrontier = clientDoc.version();

  // If the server has ops the client doesn't — normal lag, not divergence
  // If the client has ops the server doesn't — client is ahead (buffered ops not yet received by server)
  // Divergence: neither is an ancestor of the other — should be impossible with CRDT

  // Check using Loro's isAncestor — if neither is ancestor of the other, we have a problem
  const clientAheadOfServer = clientDoc.isAncestor(serverFrontier, clientFrontier);
  const serverAheadOfClient = clientDoc.isAncestor(clientFrontier, serverFrontier);

  if (!clientAheadOfServer && !serverAheadOfClient) {
    // Genuine divergence — log and trigger full resync
    logger.error('CRDT divergence detected', { scriptId, clientFrontier, serverFrontier });
    await triggerFullResync();
    return true;
  }

  return false;
}

async function triggerFullResync(): Promise<void> {
  // 1. Close the WebSocket
  // 2. Load a fresh doc from the server's crdt_snapshot
  // 3. Re-apply any local ops that are not in the server snapshot
  // 4. Reconnect
  // This is a last-resort recovery — should be extremely rare
}
```

### Redis Eviction Recovery

```typescript
// If the Redis stream for a script is evicted (TTL expired or memory pressure):
async function recoverFromRedisEviction(scriptId: string): Promise<LoroDoc> {
  logger.warn('Redis stream evicted, recovering from Postgres', { scriptId });

  // Load from the most recent crdt_snapshot in script_versions
  const row = await db.query(`
    SELECT crdt_snapshot FROM script.script_versions
    WHERE script_id = $1
    ORDER BY version_number DESC
    LIMIT 1
  `, [scriptId]);

  const doc = new LoroDoc();
  if (row.rows[0]?.crdt_snapshot) {
    doc.importSnapshot(new Uint8Array(row.rows[0].crdt_snapshot));
  } else {
    // No CRDT snapshot at all — rebuild from AST content_snapshots
    const nodes = await loadASTNodes(scriptId);
    bootstrapDocFromAST(doc, nodes);
  }

  // Warn connected clients to resync
  await redis.publish(`collab:notify:${scriptId}`, JSON.stringify({
    type: 'resync_required',
    reason: 'redis_eviction',
  }));

  return doc;
}
```

---

## 15. Testing Strategy

### Unit Tests — CRDT Properties

```typescript
describe('LoroPMBinding', () => {
  test('round-trip: PM transaction → Loro op → PM transaction', () => {
    const doc = new LoroDoc();
    const nodeId = 'test-node-uuid';
    bootstrapNodeText(doc, nodeId, 'Hello world');

    const view = createTestEditorView(doc, nodeId, 'Hello world');
    const binding = new LoroPMBinding(view, doc);

    // Apply a deletion in PM
    const tr = view.state.tr.delete(7, 12); // delete "world"
    view.dispatch(tr);

    // Verify Loro text matches
    expect(getNodeText(doc, nodeId).toString()).toBe('Hello ');
  });

  test('convergence: 3 concurrent writers, 100 random ops each', async () => {
    const scriptId = 'test-script';
    const docs = [new LoroDoc(), new LoroDoc(), new LoroDoc()];
    docs.forEach((d, i) => d.setPeerId(BigInt(i + 1)));

    // Apply random ops to each doc independently
    for (let i = 0; i < 100; i++) {
      const doc = docs[Math.floor(Math.random() * 3)];
      const text = getNodeText(doc, 'heading');
      const op = randomTextOp(text.length);
      if (op.type === 'insert') text.insert(op.pos, op.char);
      else if (op.type === 'delete' && text.length > 0) text.delete(op.pos, 1);
    }

    // Sync all docs to each other
    const updates = docs.map(d => d.exportFrom(new Uint8Array(0)));
    docs.forEach(d => updates.forEach(u => d.import(u)));

    // All docs must have converged to the same state
    const states = docs.map(d => getNodeText(d, 'heading').toString());
    expect(states[0]).toBe(states[1]);
    expect(states[1]).toBe(states[2]);
  });

  test('tree move: concurrent moves to different parents → no duplication', () => {
    const doc1 = new LoroDoc();
    const doc2 = new LoroDoc();
    doc1.setPeerId(1n);
    doc2.setPeerId(2n);

    // Bootstrap both docs with the same initial state
    const initialState = createScriptWithScene();
    doc1.import(initialState);
    doc2.import(initialState);

    // Concurrent moves to different parents
    const tree1 = doc1.getTree('structure');
    const tree2 = doc2.getTree('structure');

    tree1.move(SCENE_NODE_ID, ACT_2_NODE_ID, 1.0);
    tree2.move(SCENE_NODE_ID, ACT_3_NODE_ID, 1.0);

    // Sync
    doc1.import(doc2.exportFrom(new Uint8Array(0)));
    doc2.import(doc1.exportFrom(new Uint8Array(0)));

    // Scene must appear exactly once
    const sceneCount1 = countNodesById(doc1.getTree('structure'), SCENE_NODE_ID);
    const sceneCount2 = countNodesById(doc2.getTree('structure'), SCENE_NODE_ID);
    expect(sceneCount1).toBe(1);
    expect(sceneCount2).toBe(1);
    // Both must agree on where the scene ended up
    expect(getParentId(doc1.getTree('structure'), SCENE_NODE_ID))
      .toBe(getParentId(doc2.getTree('structure'), SCENE_NODE_ID));
  });
});
```

### Integration Tests — Full Session

```typescript
describe('Collaboration session', () => {
  test('two clients edit same scene heading, converge', async () => {
    const { client1, client2 } = await createTestClients(TEST_SCRIPT_ID);

    await client1.connect();
    await client2.connect();

    // Both start with "INT. COFFEE SHOP - DAY"
    await client1.type(HEADING_NODE_ID, 22, ' NIGHT'); // append "NIGHT"
    await client2.type(HEADING_NODE_ID, 0, 'EXT. '); // prepend "EXT. "

    // Wait for convergence (both clients receive each other's ops)
    await waitForConvergence(client1, client2);

    const text1 = await client1.getText(HEADING_NODE_ID);
    const text2 = await client2.getText(HEADING_NODE_ID);
    expect(text1).toBe(text2); // must be the same, regardless of what the text is
  });

  test('checkpoint triggers after 5 minute idle', async () => {
    const client = await createTestClient(TEST_SCRIPT_ID);
    await client.connect();
    await client.type(ACTION_NODE_ID, 0, 'Jake enters.');

    jest.advanceTimersByTime(5 * 60 * 1_000);

    const snapshot = await db.query(
      `SELECT content_snapshot FROM script.ast_nodes WHERE id = $1`,
      [ACTION_NODE_ID]
    );
    expect(snapshot.rows[0].content_snapshot).toBe('Jake enters.');
  });
});
```

### Load Test — Concurrent Editors

```typescript
// 20 concurrent editors on one document, 10 minutes of editing
describe('Load: 20 concurrent editors', () => {
  test('all editors converge, no duplicate nodes, < 5ms binding latency p90', async () => {
    const clients = await Promise.all(
      Array.from({ length: 20 }, (_, i) =>
        createTestClient(LOAD_TEST_SCRIPT_ID, `user-${i}`)
      )
    );

    await Promise.all(clients.map(c => c.connect()));

    const latencies: number[] = [];

    // Each client types 30 random characters over 10 minutes
    for (let round = 0; round < 30; round++) {
      await Promise.all(clients.map(async (client) => {
        const start = performance.now();
        await client.typeRandom();
        latencies.push(performance.now() - start);
      }));
      await sleep(20_000); // 20s between rounds → 10 minutes total
    }

    await waitForAllConvergence(clients);

    // All clients must have the same document state
    const states = await Promise.all(clients.map(c => c.getFullDocState()));
    for (let i = 1; i < states.length; i++) {
      expect(states[i]).toEqual(states[0]);
    }

    // p90 latency must be under 5ms
    latencies.sort((a, b) => a - b);
    const p90 = latencies[Math.floor(latencies.length * 0.9)];
    expect(p90).toBeLessThan(5);
  });
});
```

---

## 16. Decisions

**Loro ProseMirror binding — build in-house, contribute upstream when stable. (ADR-018)**
Binding maps ProseMirror transactions ↔ Loro ops. In-house gives us control and a ship date. Open-source contribution reduces maintenance once stable.

**Awareness protocol — custom JSON over WebSocket. (See §6)**
y-protocols is Yjs-coupled. Custom JSON protocol with the same conceptual structure (cursor position, user identity, selection range) gives us independence.

**Compaction — on revision color change (mandatory) + daily for >1,000 ops/day. (See §9)**
Revision color change is the natural clean-slate moment. Daily compaction prevents unbounded growth during long productions.

**Split/merge — decompose into atomic Loro MovableTree ops + Redis coordination lock. (ADR-019)**
No custom CRDT op. Decomposes safely into Loro's verified semantics. Lock ensures atomicity.

**Maximum concurrent editors — design target 10, hard limit 20 for v1. (See §10)**
Writers' room is 5–10 people. Beyond 20, CRDT merge complexity and fan-out degrade UX. Document splitting is the natural solution for larger groups.

**y-redis vs Hocuspocus — Hocuspocus at startup; y-redis at growth. (See §10)**
Hocuspocus is MIT; y-redis is AGPL. Defer the license and complexity until horizontal scaling is actually needed.

**Offline structural ops — not supported (require connectivity). (ADR-019)**
Structural edits (scene moves, splits, merges) require the Redis coordination lock. Text editing is fully offline-capable. This is the correct trade-off: writers do not reorganize scenes while offline on set.

**POC gate at Week 6 — hard decision point, not a soft evaluation. (See §2)**
If the 5 pass criteria are not met by Week 6, pivot to Yjs fallback immediately. Do not extend. The Yjs fallback is fully specified so the pivot is fast.
