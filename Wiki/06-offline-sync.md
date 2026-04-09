# 06 — Offline & On-Set Sync

## The Problem

Film sets have notoriously poor connectivity. Sound stages block signal, remote locations have no infrastructure, and shooting days run 12–16 hours. The Script Supervisor module must work **fully offline** for up to 7 days and sync reliably when connectivity returns — without any data loss.

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                     ON-SET (Offline-First)                          │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Tauri 2.x Desktop App (macOS / Windows)                    │   │
│  │  ┌─────────────────┐  ┌──────────────────────────────────┐ │   │
│  │  │   React UI      │  │   Rust Backend (src-tauri/)      │ │   │
│  │  │   (components   │  │   ┌─────────────┐ ┌──────────┐  │ │   │
│  │  │    shared with  │  │   │ PowerSync   │ │  Loro    │  │ │   │
│  │  │    web app)     │  │   │ Rust SDK    │ │  WASM    │  │ │   │
│  │  └────────┬────────┘  │   └──────┬──────┘ └────┬─────┘  │ │   │
│  │           │IPC        │          │              │         │ │   │
│  │           └───────────┘          ▼              ▼         │ │   │
│  │                       ┌──────────────────────────────┐    │ │   │
│  │                       │   SQLite + WAL + SQLCipher   │    │ │   │
│  │                       │   (structured on-set data)   │    │ │   │
│  │                       └──────────────────────────────┘    │ │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ When connected
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  SYNC LAYER                                         │
│  PowerSync Sync Service (cloud-managed or basecamp self-hosted)     │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────────┐ │
│  │ Sync Streams   │  │ Offline Write   │  │ Photo Upload Queue   │ │
│  │ (role-based    │  │ Queue (CRUD     │  │ (presigned S3 URLs   │ │
│  │  data routing) │  │  ops persisted) │  │  opportunistic sync) │ │
│  └────────────────┘  └─────────────────┘  └──────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│              CLOUD BACKEND (Supervisor Service)                     │
│  PostgreSQL (supervisor.*) ◄──── NATS ────► Script Service         │
│  S3 (continuity photos, PDFs)                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. SQLite Schema (On-Device)

The on-device schema mirrors the supervisor service's Postgres schema with two differences: no UUIDs v7 (SQLite uses TEXT), and no CRDT state (text fields are plain, last-writer-wins — rich text CRDT is persisted separately via Loro's Rust crate).

```sql
-- Managed by PowerSync client-side migrations

PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- ── Synced from server (read-only on device unless locally authored) ──────────

CREATE TABLE projects (
  id          TEXT PRIMARY KEY,   -- UUID
  title       TEXT NOT NULL,
  status      TEXT NOT NULL,      -- 'pre_production' | 'production' | 'post'
  updated_at  TEXT NOT NULL       -- ISO 8601
);

CREATE TABLE episodes (
  id          TEXT PRIMARY KEY,
  project_id  TEXT NOT NULL REFERENCES projects(id),
  season_num  INTEGER,
  episode_num INTEGER,
  title       TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE scenes (
  id                TEXT PRIMARY KEY,
  episode_id        TEXT NOT NULL REFERENCES episodes(id),
  scene_number      TEXT NOT NULL,   -- "42A"
  int_ext           TEXT,            -- 'INT' | 'EXT' | 'INT./EXT.'
  location_name     TEXT,
  time_of_day       TEXT,
  synopsis          TEXT,
  status            TEXT NOT NULL DEFAULT 'not_shot',   -- 'not_shot' | 'in_progress' | 'completed' | 'omitted'
  position          TEXT NOT NULL,   -- fractional index (same as AST)
  page_count        REAL,
  updated_at        TEXT NOT NULL
);

-- ── On-set writes (device-authored, synced to server) ────────────────────────

CREATE TABLE takes (
  id            TEXT PRIMARY KEY,
  scene_id      TEXT NOT NULL REFERENCES scenes(id),
  take_number   INTEGER NOT NULL,
  camera_roll   TEXT,              -- "A14"
  sound_roll    TEXT,              -- "12"
  slate_id      TEXT,              -- composite: scene + take
  setup         TEXT,              -- camera setup description
  rating        TEXT,              -- 'circle' | 'print' | 'no_print' | 'false_start'
  circle_reason TEXT,
  notes         TEXT,
  start_time    TEXT,              -- ISO 8601
  end_time      TEXT,
  created_by    TEXT NOT NULL,     -- user_id
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL,
  synced        INTEGER NOT NULL DEFAULT 0   -- 0=pending sync, 1=confirmed by server
);

CREATE TABLE deviations (
  id            TEXT PRIMARY KEY,
  scene_id      TEXT NOT NULL REFERENCES scenes(id),
  take_id       TEXT REFERENCES takes(id),
  deviation_type TEXT NOT NULL,   -- 'line_change' | 'action_change' | 'omission' | 'addition' | 'improvisation'
  original_text  TEXT,
  actual_text    TEXT,
  approved_by_director INTEGER NOT NULL DEFAULT 0,  -- boolean
  notes         TEXT,
  reported_by   TEXT NOT NULL,
  created_at    TEXT NOT NULL,
  updated_at    TEXT NOT NULL,
  synced        INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE continuity_assertions (
  id              TEXT PRIMARY KEY,
  scene_id        TEXT NOT NULL REFERENCES scenes(id),
  take_id         TEXT REFERENCES takes(id),
  story_day_id    TEXT,
  assertion_type  TEXT NOT NULL,   -- 'wardrobe' | 'hair_makeup' | 'prop' | 'set_dressing' | 'injury' | 'vfx_marker'
  subject         TEXT NOT NULL,   -- character name, prop name, etc.
  assertion_text  TEXT NOT NULL,
  photo_ids       TEXT,            -- JSON array of continuity_photo IDs
  created_by      TEXT NOT NULL,
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  synced          INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE continuity_photos (
  id              TEXT PRIMARY KEY,
  assertion_id    TEXT REFERENCES continuity_assertions(id),
  scene_id        TEXT NOT NULL REFERENCES scenes(id),
  caption         TEXT,
  local_path      TEXT NOT NULL,   -- file path on device
  s3_key          TEXT,            -- populated after upload
  upload_status   TEXT NOT NULL DEFAULT 'pending',  -- 'pending' | 'uploading' | 'done' | 'failed'
  thumbnail_path  TEXT,            -- compressed local thumbnail
  shot_at         TEXT NOT NULL,
  created_by      TEXT NOT NULL,
  created_at      TEXT NOT NULL
);

-- Loro CRDT state (binary blob — not synced via PowerSync, synced separately via WebSocket when online)
CREATE TABLE crdt_doc_states (
  doc_id       TEXT PRIMARY KEY,   -- e.g. "notes:{scene_id}"
  snapshot     BLOB NOT NULL,      -- Loro exportSnapshot() output
  vector_clock TEXT NOT NULL,      -- JSON
  updated_at   TEXT NOT NULL
);

-- Offline write queue (managed by PowerSync SDK internally, exposed here for inspection)
CREATE TABLE ps_write_queue (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  table_name   TEXT NOT NULL,
  record_id    TEXT NOT NULL,
  operation    TEXT NOT NULL,   -- 'INSERT' | 'UPDATE' | 'DELETE'
  data         TEXT NOT NULL,   -- JSON
  created_at   TEXT NOT NULL,
  retry_count  INTEGER NOT NULL DEFAULT 0,
  last_error   TEXT
);

-- Indexes
CREATE INDEX ON takes (scene_id, take_number);
CREATE INDEX ON takes (synced) WHERE synced = 0;
CREATE INDEX ON deviations (scene_id);
CREATE INDEX ON continuity_assertions (scene_id, assertion_type);
CREATE INDEX ON continuity_photos (upload_status) WHERE upload_status = 'pending';
```

---

## 3. PowerSync Sync Stream Definitions

PowerSync Sync Streams are declared server-side and control exactly what data each client role receives. Streams are defined as SQL views over the Postgres supervisor schema.

```typescript
// services/supervisor/src/powersync/sync-rules.ts
// These rules are compiled and deployed to the PowerSync service

export const SYNC_RULES = `
# ScriptOS On-Set Sync Rules
# Evaluated server-side by PowerSync — determines what reaches each device

bucket_definitions:
  # ── Project metadata (all roles see their assigned projects) ─────────────
  by_project:
    parameters: SELECT project_id FROM supervisor.project_memberships WHERE user_id = token_parameters.user_id
    data:
      - SELECT * FROM supervisor.projects WHERE id = bucket.project_id
      - SELECT * FROM supervisor.episodes WHERE project_id = bucket.project_id

  # ── Scene data + script supervisor's own takes (role-gated) ──────────────
  by_episode_supervisor:
    parameters: >
      SELECT pm.project_id, e.id AS episode_id
      FROM supervisor.project_memberships pm
      JOIN supervisor.episodes e ON e.project_id = pm.project_id
      WHERE pm.user_id = token_parameters.user_id
        AND pm.role IN ('script_supervisor', 'production_coordinator', 'showrunner')
    data:
      - SELECT * FROM supervisor.scenes WHERE episode_id = bucket.episode_id
      - SELECT * FROM supervisor.takes WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)
      - SELECT * FROM supervisor.deviations WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)
      - SELECT * FROM supervisor.continuity_assertions WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)

  # ── 1st AD sees scene status + schedule data (no takes detail) ────────────
  by_episode_ad:
    parameters: >
      SELECT pm.project_id, e.id AS episode_id
      FROM supervisor.project_memberships pm
      JOIN supervisor.episodes e ON e.project_id = pm.project_id
      WHERE pm.user_id = token_parameters.user_id
        AND pm.role IN ('first_ad', 'second_ad')
    data:
      - SELECT id, episode_id, scene_number, status, page_count, location_name, updated_at
        FROM supervisor.scenes WHERE episode_id = bucket.episode_id
      - SELECT id, scene_id, take_number, rating, start_time, end_time, updated_at
        FROM supervisor.takes WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)

  # ── Editorial sees circled takes + lined script (read-only) ───────────────
  by_episode_editorial:
    parameters: >
      SELECT pm.project_id, e.id AS episode_id
      FROM supervisor.project_memberships pm
      JOIN supervisor.episodes e ON e.project_id = pm.project_id
      WHERE pm.user_id = token_parameters.user_id
        AND pm.role IN ('editor', 'assistant_editor')
    data:
      - SELECT id, scene_id, take_number, camera_roll, sound_roll, rating, notes, updated_at
        FROM supervisor.takes
        WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)
          AND rating = 'circle'
      - SELECT * FROM supervisor.deviations WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = bucket.episode_id)
`;
```

### Postgres Schema (Supervisor Service)

```sql
-- Schema: supervisor.*
-- Authoritative server-side tables that PowerSync syncs from

CREATE TABLE supervisor.projects (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title       TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'pre_production',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE supervisor.project_memberships (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  UUID NOT NULL REFERENCES supervisor.projects(id),
  user_id     UUID NOT NULL REFERENCES auth.users(id),
  role        TEXT NOT NULL,  -- 'script_supervisor' | 'first_ad' | 'second_ad' | 'editor' | 'production_coordinator' | 'showrunner'
  UNIQUE (project_id, user_id)
);

CREATE TABLE supervisor.episodes (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id  UUID NOT NULL REFERENCES supervisor.projects(id),
  season_num  SMALLINT,
  episode_num SMALLINT,
  title       TEXT NOT NULL,
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE supervisor.scenes (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  episode_id    UUID NOT NULL REFERENCES supervisor.episodes(id),
  script_scene_id UUID,                -- FK to script.ast_nodes if linked
  scene_number  TEXT NOT NULL,
  int_ext       TEXT,
  location_name TEXT,
  time_of_day   TEXT,
  synopsis      TEXT,
  status        TEXT NOT NULL DEFAULT 'not_shot',
  position      TEXT NOT NULL,
  page_count    NUMERIC(5, 2),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE supervisor.takes (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scene_id      UUID NOT NULL REFERENCES supervisor.scenes(id),
  take_number   SMALLINT NOT NULL,
  camera_roll   TEXT,
  sound_roll    TEXT,
  setup         TEXT,
  rating        TEXT CHECK (rating IN ('circle', 'print', 'no_print', 'false_start', 'incomplete')),
  circle_reason TEXT,
  notes         TEXT,
  start_time    TIMESTAMPTZ,
  end_time      TIMESTAMPTZ,
  created_by    UUID NOT NULL REFERENCES auth.users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (scene_id, take_number)
);

CREATE TABLE supervisor.deviations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scene_id        UUID NOT NULL REFERENCES supervisor.scenes(id),
  take_id         UUID REFERENCES supervisor.takes(id),
  deviation_type  TEXT NOT NULL,
  original_text   TEXT,
  actual_text     TEXT,
  approved_by_director BOOLEAN NOT NULL DEFAULT FALSE,
  notes           TEXT,
  reported_by     UUID NOT NULL REFERENCES auth.users(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE supervisor.continuity_assertions (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  scene_id        UUID NOT NULL REFERENCES supervisor.scenes(id),
  take_id         UUID REFERENCES supervisor.takes(id),
  story_day_id    UUID REFERENCES series_timeline.timeline_story_days(id),
  assertion_type  TEXT NOT NULL,
  subject         TEXT NOT NULL,
  assertion_text  TEXT NOT NULL,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  resolved_at     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE supervisor.continuity_photos (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assertion_id    UUID REFERENCES supervisor.continuity_assertions(id),
  scene_id        UUID NOT NULL REFERENCES supervisor.scenes(id),
  caption         TEXT,
  s3_key          TEXT NOT NULL,
  thumbnail_s3_key TEXT,
  shot_at         TIMESTAMPTZ NOT NULL,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX ON supervisor.takes (scene_id, take_number);
CREATE INDEX ON supervisor.takes (rating) WHERE rating = 'circle';
CREATE INDEX ON supervisor.deviations (scene_id, take_id);
CREATE INDEX ON supervisor.continuity_assertions (scene_id, story_day_id);
```

---

## 4. Tauri IPC Protocol

The Tauri app exposes Rust commands to the React frontend via Tauri's IPC bridge. All on-set data operations go through these typed commands — never through direct fetch calls to the cloud API (which may be unavailable offline).

### Rust Command Definitions

```rust
// apps/onset/src-tauri/src/commands/mod.rs

use tauri::State;
use crate::db::SqlitePool;
use crate::powersync::PowerSyncState;
use crate::loro::LoroState;
use serde::{Deserialize, Serialize};

// ── Takes ──────────────────────────────────────────────────────────────────

#[derive(Debug, Deserialize)]
pub struct LogTakeInput {
    pub scene_id: String,
    pub take_number: u32,
    pub camera_roll: Option<String>,
    pub sound_roll: Option<String>,
    pub setup: Option<String>,
    pub notes: Option<String>,
    pub start_time: String,  // ISO 8601
}

#[derive(Debug, Serialize)]
pub struct TakeOutput {
    pub id: String,
    pub take_number: u32,
    pub rating: Option<String>,
    pub synced: bool,
}

#[tauri::command]
pub async fn log_take(
    input: LogTakeInput,
    db: State<'_, SqlitePool>,
    user_id: String,
) -> Result<TakeOutput, String> {
    let id = uuid::Uuid::now_v7().to_string();
    let now = chrono::Utc::now().to_rfc3339();

    sqlx::query!(
        r#"
        INSERT INTO takes (id, scene_id, take_number, camera_roll, sound_roll, setup, notes, start_time, created_by, created_at, updated_at, synced)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, 0)
        "#,
        id, input.scene_id, input.take_number, input.camera_roll, input.sound_roll,
        input.setup, input.notes, input.start_time, user_id, now, now
    )
    .execute(&**db)
    .await
    .map_err(|e| e.to_string())?;

    Ok(TakeOutput { id, take_number: input.take_number, rating: None, synced: false })
}

#[derive(Debug, Deserialize)]
pub struct RateTakeInput {
    pub take_id: String,
    pub rating: String,   // 'circle' | 'print' | 'no_print' | 'false_start'
    pub circle_reason: Option<String>,
    pub end_time: Option<String>,
}

#[tauri::command]
pub async fn rate_take(
    input: RateTakeInput,
    db: State<'_, SqlitePool>,
) -> Result<(), String> {
    let now = chrono::Utc::now().to_rfc3339();

    sqlx::query!(
        r#"UPDATE takes SET rating = ?, circle_reason = ?, end_time = ?, updated_at = ?, synced = 0 WHERE id = ?"#,
        input.rating, input.circle_reason, input.end_time, now, input.take_id
    )
    .execute(&**db)
    .await
    .map_err(|e| e.to_string())?;

    Ok(())
}

// ── Continuity Assertions ────────────────────────────────────────────────────

#[derive(Debug, Deserialize)]
pub struct CreateAssertionInput {
    pub scene_id: String,
    pub take_id: Option<String>,
    pub assertion_type: String,
    pub subject: String,
    pub assertion_text: String,
}

#[tauri::command]
pub async fn create_continuity_assertion(
    input: CreateAssertionInput,
    db: State<'_, SqlitePool>,
    user_id: String,
) -> Result<String, String> {
    let id = uuid::Uuid::now_v7().to_string();
    let now = chrono::Utc::now().to_rfc3339();

    sqlx::query!(
        r#"
        INSERT INTO continuity_assertions (id, scene_id, take_id, assertion_type, subject, assertion_text, created_by, created_at, updated_at, synced)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 0)
        "#,
        id, input.scene_id, input.take_id, input.assertion_type,
        input.subject, input.assertion_text, user_id, now, now
    )
    .execute(&**db)
    .await
    .map_err(|e| e.to_string())?;

    Ok(id)
}

// ── Photo Capture ────────────────────────────────────────────────────────────

#[derive(Debug, Deserialize)]
pub struct AttachPhotoInput {
    pub assertion_id: Option<String>,
    pub scene_id: String,
    pub caption: Option<String>,
    pub local_path: String,
    pub shot_at: String,
}

#[tauri::command]
pub async fn attach_continuity_photo(
    input: AttachPhotoInput,
    db: State<'_, SqlitePool>,
    user_id: String,
) -> Result<String, String> {
    let id = uuid::Uuid::now_v7().to_string();
    let now = chrono::Utc::now().to_rfc3339();

    // Generate compressed thumbnail
    let thumbnail_path = crate::image::generate_thumbnail(&input.local_path)
        .map_err(|e| e.to_string())?;

    sqlx::query!(
        r#"
        INSERT INTO continuity_photos (id, assertion_id, scene_id, caption, local_path, thumbnail_path, upload_status, shot_at, created_by, created_at)
        VALUES (?, ?, ?, ?, ?, ?, 'pending', ?, ?, ?)
        "#,
        id, input.assertion_id, input.scene_id, input.caption,
        input.local_path, thumbnail_path, input.shot_at, user_id, now
    )
    .execute(&**db)
    .await
    .map_err(|e| e.to_string())?;

    // Trigger background upload attempt (non-blocking)
    crate::photo_uploader::enqueue_upload(id.clone());

    Ok(id)
}

// ── Sync Status ──────────────────────────────────────────────────────────────

#[derive(Debug, Serialize)]
pub struct SyncStatus {
    pub connected: bool,
    pub pending_writes: u64,
    pub pending_photo_uploads: u64,
    pub last_sync_at: Option<String>,
    pub upload_progress: Option<UploadProgress>,
}

#[derive(Debug, Serialize)]
pub struct UploadProgress {
    pub uploading: String,   // photo ID currently uploading
    pub done: u64,
    pub total: u64,
}

#[tauri::command]
pub async fn get_sync_status(
    db: State<'_, SqlitePool>,
    ps: State<'_, PowerSyncState>,
) -> Result<SyncStatus, String> {
    let pending_writes: (i64,) = sqlx::query_as(
        "SELECT COUNT(*) FROM ps_write_queue"
    ).fetch_one(&**db).await.map_err(|e| e.to_string())?;

    let pending_photos: (i64,) = sqlx::query_as(
        "SELECT COUNT(*) FROM continuity_photos WHERE upload_status IN ('pending', 'failed')"
    ).fetch_one(&**db).await.map_err(|e| e.to_string())?;

    Ok(SyncStatus {
        connected: ps.is_connected(),
        pending_writes: pending_writes.0 as u64,
        pending_photo_uploads: pending_photos.0 as u64,
        last_sync_at: ps.last_sync_at().map(|t| t.to_rfc3339()),
        upload_progress: ps.upload_progress(),
    })
}

// ── Scene Status ──────────────────────────────────────────────────────────────

#[tauri::command]
pub async fn set_scene_status(
    scene_id: String,
    status: String,   // 'not_shot' | 'in_progress' | 'completed' | 'omitted'
    db: State<'_, SqlitePool>,
) -> Result<(), String> {
    let now = chrono::Utc::now().to_rfc3339();
    sqlx::query!(
        "UPDATE scenes SET status = ?, updated_at = ? WHERE id = ?",
        status, now, scene_id
    )
    .execute(&**db)
    .await
    .map_err(|e| e.to_string())?;
    Ok(())
}
```

### Frontend IPC Calls

```typescript
// apps/onset/src/api/supervisor.ts
import { invoke } from '@tauri-apps/api/core';

export const supervisorAPI = {
  logTake: (input: LogTakeInput) =>
    invoke<TakeOutput>('log_take', { input }),

  rateTake: (input: RateTakeInput) =>
    invoke<void>('rate_take', { input }),

  createAssertion: (input: CreateAssertionInput) =>
    invoke<string>('create_continuity_assertion', { input }),

  attachPhoto: (input: AttachPhotoInput) =>
    invoke<string>('attach_continuity_photo', { input }),

  getSyncStatus: () =>
    invoke<SyncStatus>('get_sync_status'),

  setSceneStatus: (sceneId: string, status: SceneStatus) =>
    invoke<void>('set_scene_status', { sceneId, status }),
};
```

---

## 5. SQLCipher Key Derivation and Management

```rust
// apps/onset/src-tauri/src/db/encryption.rs

use tauri_plugin_stronghold::Builder as StrongholdBuilder;
use argon2::{Argon2, PasswordHasher};
use rand::RngCore;

const DB_KEY_LABEL: &str = "scriptos.db_encryption_key";
const KEY_LENGTH: usize = 32; // 256 bits

/// Derive the SQLite encryption key from the authenticated session token.
/// The key is stored in the OS keychain (Stronghold) after first derivation.
/// On subsequent launches, the key is loaded from Stronghold (requires biometric unlock).
pub async fn get_or_create_db_key(
    session_token: &str,
    device_salt: &[u8; 16],
    stronghold: &tauri_plugin_stronghold::Stronghold,
) -> Result<[u8; KEY_LENGTH], crate::Error> {
    // Try to load existing key from Stronghold
    if let Ok(key) = stronghold.read(DB_KEY_LABEL).await {
        let mut buf = [0u8; KEY_LENGTH];
        buf.copy_from_slice(&key[..KEY_LENGTH]);
        return Ok(buf);
    }

    // Derive new key using Argon2id (memory-hard KDF)
    let argon2 = Argon2::default();
    let salt = argon2::password_hash::SaltString::encode_b64(device_salt)
        .map_err(|e| crate::Error::Crypto(e.to_string()))?;

    let hash = argon2
        .hash_password(session_token.as_bytes(), &salt)
        .map_err(|e| crate::Error::Crypto(e.to_string()))?;

    let mut key = [0u8; KEY_LENGTH];
    key.copy_from_slice(&hash.hash.unwrap().as_bytes()[..KEY_LENGTH]);

    // Persist to Stronghold (survives app restart, locked by device biometric)
    stronghold.write(DB_KEY_LABEL, key.to_vec()).await
        .map_err(|e| crate::Error::Stronghold(e.to_string()))?;

    Ok(key)
}

/// Called when the auth service deactivates a user (auth.user_deactivated NATS event).
/// Rotates the key — old DB data becomes unreadable until key is re-established.
pub async fn rotate_db_key(stronghold: &tauri_plugin_stronghold::Stronghold) -> Result<(), crate::Error> {
    stronghold.delete(DB_KEY_LABEL).await
        .map_err(|e| crate::Error::Stronghold(e.to_string()))?;
    Ok(())
}

/// Initialize SQLite connection with SQLCipher key applied.
pub async fn open_encrypted_db(db_path: &str, key: &[u8; KEY_LENGTH]) -> Result<sqlx::SqlitePool, crate::Error> {
    let key_hex = hex::encode(key);
    let connection_string = format!(
        "sqlite://{}?_pragma=key%3D%22x'{}'%22&_pragma=journal_mode%3DWAL&_pragma=foreign_keys%3DON",
        db_path, key_hex
    );

    sqlx::SqlitePool::connect(&connection_string)
        .await
        .map_err(|e| crate::Error::Database(e.to_string()))
}
```

---

## 6. Conflict Resolution by Data Type

### Conflict Strategy Matrix

| Data Type | Strategy | Implementation | Rationale |
|-----------|----------|---------------|-----------|
| Scene status, timecodes, take ratings | Last-Writer-Wins (LWW) register | PowerSync last-modified-wins | Latest correction by director or supervisor is authoritative |
| Take log entries | Add-wins set (union) | PowerSync INSERT-only; no DELETE | Never lose a logged take — even accidental takes are records |
| Continuity notes (rich text) | Loro text CRDT merge | Loro WASM in Tauri + WebSocket sync when online | Character-level merge preserves both sets of annotations |
| Continuity photos | Add-wins set (union) | INSERT-only; S3 keys are stable | Photos are never deleted on conflict; duplicates merged in UI |
| Camera/sound roll numbers | Max-wins | PowerSync: `MAX(camera_roll)` on conflict | Roll numbers only increment; take the highest known value |
| Scene annotations (position markers) | LWW per annotation object | PowerSync: `updated_at` timestamp ordering | Each annotation is independent; last edit wins per annotation |
| Circled take conflicts (two circles) | Manual review queue | PowerSync conflict hook → queue entry | Director judgment required — system cannot decide |
| Continuity assertion contradictions | Manual review queue | Server-side conflict detection on sync | Two assertions about the same subject/story_day may contradict |

### Conflict Detection on Server Sync

```typescript
// services/supervisor/src/conflict-detector.ts

export async function detectContinuityConflicts(
  sceneId: string,
  incomingAssertions: ContinuityAssertionRow[],
  db: DB,
): Promise<ConflictEntry[]> {
  const conflicts: ConflictEntry[] = [];

  // Find assertions for the same subject and story day that contradict each other
  for (const incoming of incomingAssertions) {
    const existing = await db.query<ContinuityAssertionRow>(sql`
      SELECT * FROM supervisor.continuity_assertions
      WHERE scene_id = ${sceneId}
        AND subject = ${incoming.subject}
        AND assertion_type = ${incoming.assertion_type}
        AND story_day_id = ${incoming.story_day_id}
        AND id != ${incoming.id}
        AND resolved_at IS NULL
    `);

    for (const ex of existing) {
      if (ex.assertion_text !== incoming.assertion_text) {
        conflicts.push({
          type: 'continuity_contradiction',
          subjectType: incoming.assertion_type,
          subject: incoming.subject,
          assertionA: { id: ex.id, text: ex.assertion_text, createdBy: ex.created_by },
          assertionB: { id: incoming.id, text: incoming.assertion_text, createdBy: incoming.created_by },
          sceneId,
        });
      }
    }
  }

  // Check for two circled takes in the same scene
  const circledTakes = await db.query<{ id: string; take_number: number }>(sql`
    SELECT id, take_number FROM supervisor.takes
    WHERE scene_id = ${sceneId} AND rating = 'circle'
    ORDER BY take_number
  `);

  if (circledTakes.length > 1) {
    conflicts.push({
      type: 'multiple_circled_takes',
      sceneId,
      circledTakeNumbers: circledTakes.map(t => t.take_number),
      requiresDirectorResolution: true,
    });
  }

  return conflicts;
}
```

---

## 7. Photo Upload Pipeline

Photos are the bandwidth bottleneck. The upload pipeline is opportunistic and bandwidth-aware.

```rust
// apps/onset/src-tauri/src/photo_uploader.rs

use tokio::sync::mpsc;
use std::path::Path;

const JPEG_QUALITY: u8 = 85;   // ~10x compression vs RAW
const THUMBNAIL_SIZE: u32 = 400; // px width

pub struct PhotoUploader {
    queue_rx: mpsc::Receiver<String>,  // photo IDs
    db: SqlitePool,
    http_client: reqwest::Client,
}

impl PhotoUploader {
    pub async fn run(&mut self) {
        while let Some(photo_id) = self.queue_rx.recv().await {
            // Check connectivity before attempting
            if !is_connected().await {
                // Re-enqueue — will retry on reconnect event
                continue;
            }

            if let Err(e) = self.upload_photo(&photo_id).await {
                log::warn!("Photo upload failed for {}: {}", photo_id, e);
                self.mark_failed(&photo_id, &e.to_string()).await;
            }
        }
    }

    async fn upload_photo(&self, photo_id: &str) -> Result<(), crate::Error> {
        let photo = sqlx::query_as::<_, PhotoRow>(
            "SELECT * FROM continuity_photos WHERE id = ?"
        )
        .bind(photo_id)
        .fetch_one(&self.db)
        .await?;

        // 1. Compress on-device
        let compressed = self.compress_image(&photo.local_path).await?;

        // 2. Get presigned upload URL from server
        let presigned = self.get_presigned_url(photo_id, compressed.len()).await?;

        // 3. Upload to S3
        self.http_client
            .put(&presigned.url)
            .header("Content-Type", "image/jpeg")
            .body(compressed)
            .send()
            .await?
            .error_for_status()?;

        // 4. Update local record with S3 key
        sqlx::query!(
            "UPDATE continuity_photos SET s3_key = ?, upload_status = 'done' WHERE id = ?",
            presigned.s3_key, photo_id
        )
        .execute(&self.db)
        .await?;

        // 5. Notify server that photo is uploaded (server updates its record)
        self.notify_server_upload_complete(photo_id, &presigned.s3_key).await?;

        Ok(())
    }

    async fn compress_image(&self, local_path: &str) -> Result<Vec<u8>, crate::Error> {
        let path = Path::new(local_path);
        let img = image::open(path)
            .map_err(|e| crate::Error::Image(e.to_string()))?;

        let mut output = Vec::new();
        let encoder = image::codecs::jpeg::JpegEncoder::new_with_quality(&mut output, JPEG_QUALITY);
        img.write_with_encoder(encoder)
            .map_err(|e| crate::Error::Image(e.to_string()))?;

        Ok(output)
    }

    async fn get_presigned_url(&self, photo_id: &str, size_bytes: usize) -> Result<PresignedUpload, crate::Error> {
        // Calls the API gateway (supervisor service) for a presigned S3 URL
        let response = self.http_client
            .post(format!("{}/supervisor/photos/{}/presign", API_BASE, photo_id))
            .json(&serde_json::json!({ "size_bytes": size_bytes, "content_type": "image/jpeg" }))
            .send()
            .await?
            .json::<PresignedUpload>()
            .await?;
        Ok(response)
    }
}

pub fn generate_thumbnail(local_path: &str) -> Result<String, crate::Error> {
    let img = image::open(local_path)
        .map_err(|e| crate::Error::Image(e.to_string()))?;

    let thumbnail = img.thumbnail(THUMBNAIL_SIZE, THUMBNAIL_SIZE * 10); // max height = 10x width
    let thumb_path = format!("{}.thumb.jpg", local_path);
    thumbnail.save_with_format(&thumb_path, image::ImageFormat::Jpeg)
        .map_err(|e| crate::Error::Image(e.to_string()))?;

    Ok(thumb_path)
}
```

---

## 8. Loro CRDT State Persistence (Rust / Tauri)

Continuity notes and deviation descriptions are rich text fields managed by Loro. Their CRDT state is persisted to the local SQLite `crdt_doc_states` table and synced via WebSocket when online (not via PowerSync — PowerSync is for structured data only).

```rust
// apps/onset/src-tauri/src/loro.rs

use loro::{LoroDoc, VersionVector};
use sqlx::SqlitePool;

pub struct LoroState {
    docs: std::collections::HashMap<String, LoroDoc>,
    db: SqlitePool,
}

impl LoroState {
    /// Load CRDT doc state from SQLite — called on app launch
    pub async fn load_doc(&mut self, doc_id: &str) -> Result<&LoroDoc, crate::Error> {
        if self.docs.contains_key(doc_id) {
            return Ok(&self.docs[doc_id]);
        }

        let doc = LoroDoc::new();

        let row = sqlx::query!(
            "SELECT snapshot FROM crdt_doc_states WHERE doc_id = ?",
            doc_id
        )
        .fetch_optional(&self.db)
        .await?;

        if let Some(row) = row {
            doc.import(&row.snapshot)
                .map_err(|e| crate::Error::Crdt(e.to_string()))?;
        }

        self.docs.insert(doc_id.to_string(), doc);
        Ok(&self.docs[doc_id])
    }

    /// Persist CRDT state to SQLite after every local op
    pub async fn save_doc(&self, doc_id: &str) -> Result<(), crate::Error> {
        let doc = self.docs.get(doc_id)
            .ok_or_else(|| crate::Error::Crdt(format!("Doc {} not loaded", doc_id)))?;

        let snapshot = doc.export_snapshot();
        let vv = serde_json::to_string(&doc.oplog_vv())
            .map_err(|e| crate::Error::Serialization(e.to_string()))?;
        let now = chrono::Utc::now().to_rfc3339();

        sqlx::query!(
            r#"
            INSERT INTO crdt_doc_states (doc_id, snapshot, vector_clock, updated_at)
            VALUES (?, ?, ?, ?)
            ON CONFLICT (doc_id) DO UPDATE SET
              snapshot = excluded.snapshot,
              vector_clock = excluded.vector_clock,
              updated_at = excluded.updated_at
            "#,
            doc_id, snapshot, vv, now
        )
        .execute(&self.db)
        .await?;

        Ok(())
    }

    /// Called when WebSocket sync arrives — merge remote ops into local doc
    pub async fn apply_remote_update(&mut self, doc_id: &str, update: &[u8]) -> Result<(), crate::Error> {
        let doc = self.docs.get_mut(doc_id)
            .ok_or_else(|| crate::Error::Crdt(format!("Doc {} not loaded", doc_id)))?;

        doc.import(update)
            .map_err(|e| crate::Error::Crdt(e.to_string()))?;

        self.save_doc(doc_id).await?;
        Ok(())
    }

    /// Export ops since the given version vector (for sync protocol SyncStep1)
    pub fn export_since(&self, doc_id: &str, since: &VersionVector) -> Result<Vec<u8>, crate::Error> {
        let doc = self.docs.get(doc_id)
            .ok_or_else(|| crate::Error::Crdt(format!("Doc {} not loaded", doc_id)))?;

        doc.export_from(since)
            .map_err(|e| crate::Error::Crdt(e.to_string()))
    }
}
```

---

## 9. PowerSync Connection Lifecycle

```rust
// apps/onset/src-tauri/src/powersync.rs

use powersync_sdk::PowerSync;

pub struct PowerSyncState {
    client: PowerSync,
    connected: std::sync::atomic::AtomicBool,
    last_sync_at: std::sync::Mutex<Option<chrono::DateTime<chrono::Utc>>>,
}

impl PowerSyncState {
    pub async fn new(db_path: &str, db_key: &[u8; 32]) -> Result<Self, crate::Error> {
        let client = PowerSync::builder()
            .database_path(db_path)
            .database_key(db_key)
            .endpoint(
                std::env::var("POWERSYNC_ENDPOINT")
                    .unwrap_or_else(|_| "https://sync.scriptos.io".into())
            )
            // Basecamp fallback: if env var set, prefer local sync server
            .fallback_endpoint(std::env::var("POWERSYNC_LOCAL_ENDPOINT").ok())
            .build()
            .await?;

        Ok(Self {
            client,
            connected: std::sync::atomic::AtomicBool::new(false),
            last_sync_at: std::sync::Mutex::new(None),
        })
    }

    /// Start sync and listen to connectivity events
    pub async fn start_sync(&self, auth_token: &str) -> Result<(), crate::Error> {
        self.client.connect(auth_token).await?;

        // Listen for sync events
        let mut events = self.client.subscribe_events();
        let connected = &self.connected;
        let last_sync_at = &self.last_sync_at;

        tokio::spawn(async move {
            while let Some(event) = events.recv().await {
                match event {
                    powersync_sdk::Event::Connected => {
                        connected.store(true, std::sync::atomic::Ordering::SeqCst);
                        log::info!("PowerSync connected");
                    }
                    powersync_sdk::Event::Disconnected { reason } => {
                        connected.store(false, std::sync::atomic::Ordering::SeqCst);
                        log::warn!("PowerSync disconnected: {}", reason);
                    }
                    powersync_sdk::Event::SyncComplete { at } => {
                        *last_sync_at.lock().unwrap() = Some(at);
                        log::info!("Sync complete at {}", at);
                    }
                    powersync_sdk::Event::UploadError { error } => {
                        log::error!("Upload error: {}", error);
                    }
                }
            }
        });

        Ok(())
    }

    pub fn is_connected(&self) -> bool {
        self.connected.load(std::sync::atomic::Ordering::SeqCst)
    }

    pub fn last_sync_at(&self) -> Option<chrono::DateTime<chrono::Utc>> {
        *self.last_sync_at.lock().unwrap()
    }
}
```

---

## 10. Basecamp Self-Hosted Deployment

For remote locations without internet, a self-hosted PowerSync instance runs on a production laptop at basecamp.

```yaml
# docker-compose.basecamp.yml
# Deployed to production basecamp laptop — no internet required after initial pull

services:
  postgres:
    image: postgres:16
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./basecamp-seed.sql:/docker-entrypoint-initdb.d/seed.sql
    environment:
      POSTGRES_DB: scriptos_basecamp
      POSTGRES_PASSWORD: ${BASECAMP_PG_PASSWORD}
    healthcheck:
      test: pg_isready -U postgres

  powersync:
    image: journeyapps/powersync-service:latest
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      POWERSYNC_CONFIG: |
        replication:
          connections:
            - type: postgresql
              uri: postgresql://postgres:${BASECAMP_PG_PASSWORD}@postgres:5432/scriptos_basecamp
        client_auth:
          supabase_jwt_secret: ${BASECAMP_JWT_SECRET}
    ports:
      - "8080:8080"    # On local WiFi network — devices connect to laptop IP:8080
    volumes:
      - powersync_data:/data

volumes:
  postgres_data:
  powersync_data:
```

```bash
# scripts/basecamp-setup.sh
# Run once before leaving for a remote location

# 1. Pull latest data from cloud Postgres for assigned project
pg_dump \
  "postgresql://scriptos:${CLOUD_PG_PASSWORD}@${CLOUD_PG_HOST}/scriptos" \
  --schema=supervisor \
  --table="supervisor.projects" \
  --table="supervisor.episodes" \
  --table="supervisor.scenes" \
  --where="project_id = '${PROJECT_ID}'" \
  > basecamp-seed.sql

# 2. Start local services
docker compose -f docker-compose.basecamp.yml up -d

# 3. Print local WiFi IP for device configuration
echo "Basecamp sync endpoint: http://$(hostname -I | awk '{print $1}'):8080"
echo "Configure on-set devices to use this endpoint"
```

---

## 11. Sync Resume After Offline Period

When the Tauri app reconnects (or the basecamp laptop reaches internet), PowerSync automatically flushes the offline write queue. The sync resumes without user action.

### NATS Events Published on Sync

When structured data syncs to the cloud Postgres, the Supervisor Service publishes NATS events so downstream services can react:

```typescript
// services/supervisor/src/sync-event-publisher.ts

// Called by a Postgres trigger or Debezium CDC consumer when synced rows arrive
export async function publishSyncEvents(
  newRows: SyncedRow[],
  nats: NatsClient,
): Promise<void> {
  for (const row of newRows) {
    switch (row.table) {
      case 'supervisor.takes':
        if (row.data.rating === 'circle') {
          // Publish to editorial — circled take is editorial-relevant
          await nats.publish('supervisor.take.circled', {
            takeId: row.data.id,
            sceneId: row.data.scene_id,
            takeNumber: row.data.take_number,
            cameraRoll: row.data.camera_roll,
            projectId: row.data.project_id,
          });
        }
        break;

      case 'supervisor.scenes':
        if (row.data.status === 'completed') {
          await nats.publish('supervisor.scene.completed', {
            sceneId: row.data.id,
            episodeId: row.data.episode_id,
            sceneNumber: row.data.scene_number,
          });
          // Trigger script supervisor day-end turnover generation
          await nats.publish('supervisor.turnover.requested', {
            sceneId: row.data.id,
          });
        }
        break;

      case 'supervisor.continuity_assertions':
        // Notify continuity service to update SeriesTimeline
        await nats.publish('supervisor.assertion.synced', {
          assertionId: row.data.id,
          storyDayId: row.data.story_day_id,
          subject: row.data.subject,
          assertionType: row.data.assertion_type,
          assertionText: row.data.assertion_text,
        });
        break;
    }
  }
}
```

---

## 12. Testing Strategy

### Unit Tests (Rust)

```rust
// apps/onset/src-tauri/src/__tests__/commands_test.rs
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx::SqlitePool;

    async fn setup_test_db() -> SqlitePool {
        let pool = SqlitePool::connect(":memory:").await.unwrap();
        sqlx::migrate!("./migrations").run(&pool).await.unwrap();
        pool
    }

    #[tokio::test]
    async fn test_log_take_persists_with_synced_false() {
        let db = setup_test_db().await;
        let result = log_take(
            LogTakeInput { scene_id: "scene-1".into(), take_number: 1, camera_roll: Some("A01".into()), ..Default::default() },
            tauri::State::new(db.clone()),
            "user-1".into(),
        ).await.unwrap();

        assert!(!result.synced);
        assert_eq!(result.take_number, 1);

        let row: (i64,) = sqlx::query_as("SELECT synced FROM takes WHERE id = ?")
            .bind(&result.id).fetch_one(&db).await.unwrap();
        assert_eq!(row.0, 0); // not yet synced
    }

    #[tokio::test]
    async fn test_photo_thumbnail_generated() {
        let thumb = generate_thumbnail("fixtures/test_photo.jpg").unwrap();
        assert!(std::path::Path::new(&thumb).exists());
        let metadata = std::fs::metadata(&thumb).unwrap();
        assert!(metadata.len() < 100_000); // thumbnail < 100KB
    }
}
```

### Integration Tests — 72-Hour Offline Simulation

```typescript
// services/supervisor/src/__tests__/integration/offline-sync.test.ts
describe('72-hour offline simulation', () => {
  it('queues 8 shooting days of data and syncs without data loss', async () => {
    // Simulate: disconnect PowerSync, generate 8 days of on-set data,
    // reconnect, assert all records reached cloud Postgres

    const { db, ps } = await setupTestEnvironment({ startConnected: true });
    await ps.disconnect();

    const expected = {
      takes: 0, scenes: 0, assertions: 0, deviations: 0,
    };

    // Generate 8 simulated shooting days
    for (let day = 0; day < 8; day++) {
      const sceneIds = await generateShootingDay(db, day);
      for (const sceneId of sceneIds) {
        const numTakes = 5 + Math.floor(Math.random() * 15);
        for (let t = 1; t <= numTakes; t++) {
          await db.run('INSERT INTO takes ...', [sceneId, t]);
          expected.takes++;
        }
        expected.scenes++;
        // Add continuity assertions and deviations
        expected.assertions += await addContinuityData(db, sceneId);
        expected.deviations += await addDeviations(db, sceneId);
      }
    }

    // Reconnect and wait for sync
    await ps.connect(AUTH_TOKEN);
    await ps.waitForSyncComplete({ timeoutMs: 30_000 });

    // Assert all records present in cloud Postgres
    const cloudTakes = await cloudDb.queryOne<{ count: number }>(sql`SELECT COUNT(*) as count FROM supervisor.takes WHERE project_id = ${TEST_PROJECT_ID}`);
    expect(cloudTakes.count).toBe(expected.takes);

    const cloudAssertions = await cloudDb.queryOne<{ count: number }>(sql`SELECT COUNT(*) as count FROM supervisor.continuity_assertions WHERE scene_id IN (SELECT id FROM supervisor.scenes WHERE episode_id = ${TEST_EPISODE_ID})`);
    expect(cloudAssertions.count).toBe(expected.assertions);
  });

  it('resolves circled take conflict with manual review queue entry', async () => {
    const sceneId = await createTestScene();

    // Two devices offline, both circle different takes
    await deviceA.rateTake({ takeId: 'take-1', rating: 'circle' });
    await deviceB.rateTake({ takeId: 'take-2', rating: 'circle' });

    // Both reconnect
    await Promise.all([deviceA.sync(), deviceB.sync()]);

    // Conflict detector should flag this
    const conflicts = await cloudDb.query(sql`
      SELECT * FROM supervisor.sync_conflicts WHERE scene_id = ${sceneId}
    `);
    expect(conflicts).toHaveLength(1);
    expect(conflicts[0].type).toBe('multiple_circled_takes');
  });
});
```

---

## Decisions

**PowerSync licensing for self-hosted basecamp — open source (Apache 2.0), no cost.**
PowerSync's self-hosted server is open source under Apache 2.0. A production laptop running PowerSync locally at a remote basecamp is a legitimate open-source use — no commercial license required. The commercial relationship with PowerSync is for the managed cloud sync service used in normal connected operation. Self-hosted basecamp is a contingency deployment, not a primary one.

**Photo sync strategy — compress on-device to JPEG 85% before upload.**
On-device compression reduces a 12MP continuity photo from ~5MB to ~500KB (10x bandwidth saving on mobile/satellite networks). The compressed JPEG is uploaded to S3 via presigned URL when connectivity is available. The original full-resolution image remains on the device's camera roll for the duration of the shoot — accessible if higher resolution is needed. Metadata (S3 key, scene ref, story day) syncs via PowerSync independently of the photo upload.

**Tauri 2.x iOS — desktop (macOS/Windows) at launch; iOS evaluated Q3 2026.**
Tauri 2.x desktop is mature and is the primary platform for Script Supervisors (laptop on set). Tauri 2.x iOS support was released in 2024 and is newer than desktop. Ship desktop at launch. Evaluate iOS readiness in Q3 2026; if Tauri iOS proves unready for production, ship a React Native wrapper sharing the same React component library. The UI code (React/TypeScript) is shared regardless of native wrapper, so the fallback is low-cost.

**SQLCipher key management — OS device keychain (Stronghold), derived from session token via Argon2id.**
Key derived from the user's session token using Argon2id (memory-hard KDF), stored in the OS keychain via Tauri Stronghold plugin (macOS Keychain, Windows Credential Manager, iOS Secure Enclave). Unlocked automatically by device biometric/PIN — no additional passphrase. Key is invalidated on logout or when the Auth Service deactivates the user (`auth.user_deactivated` NATS event triggers key rotation on next sync). User passphrase rejected: Script Supervisors cannot afford friction unlocking their device on set. Org-managed keys rejected: require key distribution infrastructure that doesn't work offline.

**Maximum offline duration — test target 72 hours, design target 7 days.**
72 hours covers the worst-case contiguous remote shoot (Friday shoot to Monday sync). 7-day design target covers remote location shoots (jungle, desert, international locations without reliable connectivity). SQLite WAL mode and PowerSync's offline queue handle this duration without issue. Test suite must include a simulated 72-hour offline → sync scenario to validate queue size, conflict counts, and sync completion time.

**Conflict strategy — LWW for scalar fields, union for log entries, CRDT for rich text, manual review for human-judgment conflicts.**
Scalar on-set data (take ratings, scene status) is corrected in place — LWW is correct because the latest correction is always authoritative. Log entries (takes, photos, assertions) are append-only records — union semantics never lose data. Rich text (continuity notes) benefits from character-level CRDT merge to preserve concurrent annotations. Circled take conflicts and continuity contradictions require director/supervisor judgment — surface in a review queue rather than making an automatic decision.

**PowerSync Sync Streams — role-based, episode-granularity.**
Script supervisors receive all structured data for their assigned episodes. 1st ADs receive scene status and take summaries (not full take detail). Editorial receives only circled takes and deviations. This minimizes SQLite database size on each device (reduces risk of storage exhaustion), minimizes sync bandwidth, and enforces data access control at the sync layer rather than relying solely on application logic.

**Loro CRDT state — persisted to SQLite separately from PowerSync queue.**
PowerSync manages structured relational data (takes, deviations, assertions). Loro manages rich text CRDT state (continuity notes). These have different sync mechanisms: PowerSync uses Postgres replication for structured data; Loro uses the WebSocket collaboration protocol when online. Keeping them separate avoids conflating two distinct sync channels and allows each to operate optimally. When offline, both channels degrade gracefully: PowerSync queues writes to SQLite; Loro persists snapshots to the local `crdt_doc_states` table and syncs via WebSocket when connectivity returns.
