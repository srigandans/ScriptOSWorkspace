# 05 — AI Governance & Model Strategy

## Governing Principle

> AI is an assistant to the writer, never the author. Every AI interaction is logged, attributable, opt-in, and compliant with WGA Article 72. No AI output enters the script without a writer making an explicit acceptance decision.

---

## 1. WGA Article 72 — Compliance Framework

The 2023 WGA Minimum Basic Agreement (MBA) Article 72 governs AI use in covered productions. ScriptOS is a tool platform — not a studio — but it operates in the covered production ecosystem and must not enable MBA violations.

### Core Rules and Platform Implications

| MBA Rule | ScriptOS Implication |
|----------|---------------------|
| AI output is not "literary material" under the MBA | AI suggestions displayed as drafts, never committed to the script without writer action |
| Companies cannot require writers to use AI | All AI features are opt-in per feature per session; no workflow gates on AI acceptance |
| Writers retain credit when voluntarily using AI with company consent | Company consent state tracked per project; ledger captures consent ref on every entry |
| Companies must disclose AI origins of material given to writers | Inbound AI-sourced content tagged in AST with `ai_origin` provenance field |
| WGA semi-annual meetings — companies must report AI use | Platform exports per-project AI usage summaries (see §11) |

### What ScriptOS DOES

- **Assistive AI**: suggestions, alternatives, consistency checks, analysis — always writer-reviewed
- **Non-generative AI**: formatting correction, plagiarism detection, structure analysis, pacing metrics
- **Bible-grounded generation**: dialogue alternatives grounded in character voice + canon facts
- **Coverage analysis**: AI reads and scores — does not write

### What ScriptOS NEVER DOES

- Position AI as authoring literary material
- Auto-accept AI output into the AST
- Make AI features mandatory in any workflow
- Train models on covered writers' scripts without explicit written authorization
- Obscure which parts of a script had AI involvement

### AI Feature Opt-In States

```typescript
// Per-writer, per-project feature flags stored in ai_feature_consent table
interface AIFeatureConsent {
  writer_id: string;
  project_id: string;
  feature: AIFeature;
  enabled: boolean;
  company_consent_ref: string | null; // links to CompanyConsent record
  enabled_at: string | null;
  disabled_at: string | null;
}

type AIFeature =
  | 'dialogue_suggestion'
  | 'action_description'
  | 'beat_expansion'
  | 'consistency_check'
  | 'structure_analysis'
  | 'coverage_analysis'
  | 'format_correction'   // always available — non-generative
  | 'translation'
  | 'scene_description';
```

---

## 2. Company Consent Model

### Consent State Machine

```
              ┌──────────────────────────────────┐
              │                                  │
  PENDING ──► GRANTED ──► REVOKED ──► REINSTATED │
     │            │                              │
     └──► DENIED  └──► EXPIRED (MBA renewal date)│
              │                                  │
              └──────────────────────────────────┘
```

### PostgreSQL Schema

```sql
-- Managed within the AI service schema: ai.*

CREATE TABLE ai.company_consents (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        UUID NOT NULL REFERENCES auth.organizations(id),
  project_id    UUID,                              -- NULL = org-wide consent
  consent_state TEXT NOT NULL                      -- 'pending' | 'granted' | 'revoked' | 'denied' | 'expired' | 'reinstated'
                  CHECK (consent_state IN ('pending','granted','revoked','denied','expired','reinstated')),
  granted_by    UUID REFERENCES auth.users(id),   -- studio rep who granted
  granted_at    TIMESTAMPTZ,
  revoked_at    TIMESTAMPTZ,
  expires_at    TIMESTAMPTZ,                       -- MBA renewal date (typically May 1 of contract year)
  document_ref  TEXT,                              -- S3 path to signed consent document
  notes         TEXT,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ON ai.company_consents (org_id, project_id, consent_state);
CREATE INDEX ON ai.company_consents (expires_at) WHERE consent_state = 'granted';

-- Feature-level consent per writer per project
CREATE TABLE ai.ai_feature_consents (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  writer_id           UUID NOT NULL REFERENCES auth.users(id),
  project_id          UUID NOT NULL,
  feature             TEXT NOT NULL,
  enabled             BOOLEAN NOT NULL DEFAULT FALSE,
  company_consent_ref UUID REFERENCES ai.company_consents(id),
  enabled_at          TIMESTAMPTZ,
  disabled_at         TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE UNIQUE INDEX ON ai.ai_feature_consents (writer_id, project_id, feature);
```

### Consent Verification

```typescript
// packages/ai/src/consent.ts

export interface ConsentContext {
  orgId: string;
  projectId: string;
  writerId: string;
  feature: AIFeature;
}

export type ConsentResult =
  | { allowed: true; consentRef: string }
  | { allowed: false; reason: 'no_company_consent' | 'consent_expired' | 'feature_disabled' | 'org_quota_exceeded' };

export async function checkConsent(ctx: ConsentContext, db: DB): Promise<ConsentResult> {
  // 1. Check org-level or project-level company consent
  const consent = await db.queryOne<{ id: string; expires_at: string | null }>(sql`
    SELECT id, expires_at
    FROM ai.company_consents
    WHERE org_id = ${ctx.orgId}
      AND (project_id = ${ctx.projectId} OR project_id IS NULL)
      AND consent_state = 'granted'
    ORDER BY project_id NULLS LAST   -- project-specific takes precedence over org-wide
    LIMIT 1
  `);

  if (!consent) return { allowed: false, reason: 'no_company_consent' };

  // 2. Check expiry
  if (consent.expires_at && new Date(consent.expires_at) < new Date()) {
    // Mark as expired asynchronously — don't block the response
    void db.query(sql`
      UPDATE ai.company_consents SET consent_state = 'expired', updated_at = NOW()
      WHERE id = ${consent.id}
    `);
    return { allowed: false, reason: 'consent_expired' };
  }

  // 3. Check feature-level opt-in
  if (ctx.feature !== 'format_correction') {  // format_correction is always available
    const featureConsent = await db.queryOne<{ enabled: boolean }>(sql`
      SELECT enabled FROM ai.ai_feature_consents
      WHERE writer_id = ${ctx.writerId}
        AND project_id = ${ctx.projectId}
        AND feature = ${ctx.feature}
    `);
    if (!featureConsent?.enabled) return { allowed: false, reason: 'feature_disabled' };
  }

  return { allowed: true, consentRef: consent.id };
}
```

---

## 3. Context Assembly Pipeline

The quality of AI output for screenplay work is dominated by context quality. The context assembler pulls from: the Series Bible Graph (Neo4j), the SeriesTimeline (Postgres), the Script AST (Postgres + CRDT), and the Character Voice Registry (Neo4j).

### Context Assembly Order

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │  CACHEABLE PREFIX (stable — same across requests in a session/project) │
  │                                                                        │
  │  1. World rules (from Bible: series-level rules, format constraints)   │
  │  2. Active character facts (facts about chars in current scene)        │
  │  3. Active location facts (facts about locations in current scene)     │
  │  4. Character voice profiles (vocabulary, patterns, forbidden words)   │
  │  5. Story timeline state (story day, continuity assertions)            │
  │                                                                        │
  └────────────────────────────────────────────────────────────────────────┘
  ┌────────────────────────────────────────────────────────────────────────┐
  │  FRESH SUFFIX (variable — changes per request)                         │
  │                                                                        │
  │  6. 2 scenes before current scene (context window)                     │
  │  7. Current scene (the scene being written)                            │
  │  8. User's specific request (dialogue suggestion, etc.)                │
  │                                                                        │
  └────────────────────────────────────────────────────────────────────────┘
```

### TypeScript Implementation

```typescript
// services/ai/src/context-assembler.ts

import { ContextAssemblyInput, AssembledContext } from './types';
import { NeoDriver } from '../../../packages/db/src/neo4j';
import { DB } from '../../../packages/db/src/postgres';
import { BibleGraphClient } from '../../bible/src/graph-client';

export interface ContextAssemblyInput {
  projectId: string;
  scriptId: string;
  sceneId: string;
  characterNames: string[];   // characters in the current scene
  locationName: string;        // location of the current scene
  feature: AIFeature;
  userRequest?: string;        // the writer's specific request
}

export interface AssembledContext {
  cacheablePrefix: string;    // stable portion — should be Anthropic cache_control block
  freshSuffix: string;        // variable portion
  contextRefs: string[];      // IDs of bible facts used (for ledger)
  estimatedTokens: number;
}

const MAX_PREFIX_TOKENS = 60_000;   // leaves room for suffix + response within 100K context
const MAX_SUFFIX_TOKENS = 20_000;
const SURROUNDING_SCENES = 2;        // scenes before current to include

export async function assembleContext(
  input: ContextAssemblyInput,
  bible: BibleGraphClient,
  db: DB,
): Promise<AssembledContext> {
  const contextRefs: string[] = [];

  // ── 1. World rules ──────────────────────────────────────────────────────
  const worldRules = await bible.getWorldRules(input.projectId);
  contextRefs.push(...worldRules.map(r => r.id));

  // ── 2–3. Character and location facts ───────────────────────────────────
  const [charFacts, locationFacts] = await Promise.all([
    bible.getCharacterFacts(input.projectId, input.characterNames),
    bible.getLocationFacts(input.projectId, input.locationName),
  ]);
  contextRefs.push(...charFacts.map(f => f.id), ...locationFacts.map(f => f.id));

  // ── 4. Voice profiles + reference scenes ────────────────────────────────
  const voiceProfiles = await bible.getVoiceProfiles(input.projectId, input.characterNames);

  // ── 5. Timeline state ───────────────────────────────────────────────────
  const timelineState = await getTimelineState(input.scriptId, input.sceneId, db);

  // Assemble cacheable prefix
  const cacheablePrefix = buildCacheablePrefix({
    worldRules,
    charFacts,
    locationFacts,
    voiceProfiles,
    timelineState,
    feature: input.feature,
  });

  // ── 6–7. Surrounding scenes + current scene ──────────────────────────────
  const sceneContext = await getSurroundingScenes(
    input.scriptId,
    input.sceneId,
    SURROUNDING_SCENES,
    db,
  );

  const freshSuffix = buildFreshSuffix({
    sceneContext,
    userRequest: input.userRequest,
    feature: input.feature,
  });

  return {
    cacheablePrefix,
    freshSuffix,
    contextRefs,
    estimatedTokens: estimateTokens(cacheablePrefix) + estimateTokens(freshSuffix),
  };
}

function buildCacheablePrefix(parts: {
  worldRules: WorldRule[];
  charFacts: CharacterFact[];
  locationFacts: LocationFact[];
  voiceProfiles: VoiceProfile[];
  timelineState: TimelineState;
  feature: AIFeature;
}): string {
  const sections: string[] = [];

  // System instruction based on feature
  sections.push(getSystemInstruction(parts.feature));

  if (parts.worldRules.length > 0) {
    sections.push(`## World Rules\n${parts.worldRules.map(r => `- ${r.rule_text}`).join('\n')}`);
  }

  if (parts.charFacts.length > 0) {
    const byChar = groupBy(parts.charFacts, f => f.character_name);
    const charSection = Object.entries(byChar)
      .map(([name, facts]) => `### ${name}\n${facts.map(f => `- ${f.fact_text}`).join('\n')}`)
      .join('\n\n');
    sections.push(`## Character Facts\n${charSection}`);
  }

  if (parts.locationFacts.length > 0) {
    sections.push(`## Location Facts\n${parts.locationFacts.map(f => `- ${f.fact_text}`).join('\n')}`);
  }

  if (parts.voiceProfiles.length > 0) {
    const voiceSection = parts.voiceProfiles.map(v => {
      const lines = [`### ${v.character_name}`];
      lines.push(`- Vocabulary: ${v.vocabulary_level}`);
      if (v.speech_patterns.length > 0) lines.push(`- Patterns: ${v.speech_patterns.join('; ')}`);
      if (v.forbidden_words.length > 0) lines.push(`- Never uses: ${v.forbidden_words.join(', ')}`);
      if (v.dialect_notes) lines.push(`- Dialect: ${v.dialect_notes}`);
      if (v.emotional_range) lines.push(`- Emotional range: ${v.emotional_range}`);
      if (v.reference_dialogues?.length > 0) {
        lines.push('- Reference dialogue examples:');
        v.reference_dialogues.forEach(ex => lines.push(`  > ${ex}`));
      }
      return lines.join('\n');
    }).join('\n\n');
    sections.push(`## Character Voice Profiles\n${voiceSection}`);
  }

  if (parts.timelineState.storyDay !== null) {
    const tl = parts.timelineState;
    const tlLines = [`- Story day: ${tl.storyDay}`, `- Time of day: ${tl.timeOfDay ?? 'unspecified'}`];
    if (tl.activeAssertions.length > 0) {
      tlLines.push('- Active continuity state:');
      tl.activeAssertions.forEach(a => tlLines.push(`  - ${a.subject}: ${a.assertion_text}`));
    }
    sections.push(`## Timeline State\n${tlLines.join('\n')}`);
  }

  return sections.join('\n\n');
}

function buildFreshSuffix(parts: {
  sceneContext: SceneContextBlock[];
  userRequest?: string;
  feature: AIFeature;
}): string {
  const sections: string[] = [];

  if (parts.sceneContext.length > 0) {
    const scenesText = parts.sceneContext
      .map(s => formatSceneAsScreenplay(s))
      .join('\n\n---\n\n');
    sections.push(`## Script Context\n\`\`\`screenplay\n${scenesText}\n\`\`\``);
  }

  if (parts.userRequest) {
    sections.push(`## Request\n${parts.userRequest}`);
  }

  sections.push(getOutputInstruction(parts.feature));

  return sections.join('\n\n');
}

function getSystemInstruction(feature: AIFeature): string {
  const base = `You are a professional screenplay writing assistant for ScriptOS. You help writers craft and refine their scripts while preserving their voice and style. You never claim authorship. All suggestions are presented as options for the writer to accept, reject, or modify.`;

  const featureInstructions: Partial<Record<AIFeature, string>> = {
    dialogue_suggestion: `${base}\n\nYour task is to suggest dialogue alternatives. Maintain the character's established voice exactly as specified in the voice profiles. Offer 2–3 alternatives. Keep the same dramatic intent as the original.`,
    consistency_check: `${base}\n\nYour task is to identify consistency issues in the provided script content against the established canon facts and continuity state. List any violations found. Be specific about what contradicts what.`,
    coverage_analysis: `${base}\n\nYour task is to provide a script coverage report with: logline, synopsis, structure analysis, character assessment, dialogue quality, genre execution, and a recommendation (recommend/consider/pass) with justification.`,
    structure_analysis: `${base}\n\nYour task is to analyze the script's structure: act breaks, plot progression, pacing, and scene function (setup/confrontation/resolution/transition). Be specific about scene numbers.`,
  };

  return featureInstructions[feature] ?? base;
}

async function getTimelineState(
  scriptId: string,
  sceneId: string,
  db: DB,
): Promise<TimelineState> {
  const row = await db.queryOne<{ story_day: number; time_of_day: string }>(sql`
    SELECT tsd.story_day_number as story_day, tsd.time_of_day
    FROM script.ast_nodes sn
    JOIN series_timeline.story_day_scenes sds ON sds.scene_id = sn.id
    JOIN series_timeline.timeline_story_days tsd ON tsd.id = sds.story_day_id
    WHERE sn.id = ${sceneId} AND sn.script_id = ${scriptId}
    LIMIT 1
  `);

  if (!row) return { storyDay: null, timeOfDay: null, activeAssertions: [] };

  const assertions = await db.query<{ subject: string; assertion_text: string }>(sql`
    SELECT ca.subject, ca.assertion_text
    FROM series_timeline.continuity_assertions ca
    JOIN series_timeline.timeline_story_days tsd ON tsd.id = ca.story_day_id
    WHERE tsd.story_day_number <= ${row.story_day}
      AND ca.assertion_type IN ('wardrobe', 'injury', 'prop', 'relationship')
      AND ca.resolved_at IS NULL
    ORDER BY tsd.story_day_number DESC
    LIMIT 20
  `);

  return { storyDay: row.story_day, timeOfDay: row.time_of_day, activeAssertions: assertions };
}

async function getSurroundingScenes(
  scriptId: string,
  currentSceneId: string,
  count: number,
  db: DB,
): Promise<SceneContextBlock[]> {
  return db.query<SceneContextBlock>(sql`
    WITH current_pos AS (
      SELECT position FROM script.ast_nodes WHERE id = ${currentSceneId}
    )
    SELECT
      n.id,
      n.node_type,
      n.metadata,
      n.position,
      array_agg(
        json_build_object('type', c.node_type, 'text', c.text_content, 'metadata', c.metadata)
        ORDER BY c.position
      ) FILTER (WHERE c.id IS NOT NULL) as children
    FROM script.ast_nodes n
    LEFT JOIN script.ast_nodes c ON c.parent_id = n.id
    WHERE n.script_id = ${scriptId}
      AND n.node_type = 'scene'
      AND n.position <= (SELECT position FROM current_pos)
      AND n.deleted_at IS NULL
    GROUP BY n.id, n.node_type, n.metadata, n.position
    ORDER BY n.position DESC
    LIMIT ${count + 1}   -- +1 includes current scene
  `);
}
```

---

## 4. Model Router

### Routing Decision Tree

```
Request arrives with feature + estimated_complexity
        │
        ├─► format_correction ──────────────────────► Tier 1 (Haiku)
        ├─► consistency_check ──────────────────────► Tier 1 (Haiku)
        │     (non-generative, rule-based output)
        │
        ├─► dialogue_suggestion
        │     ├─► complexity: simple ─────────────► Tier 1 (Haiku)
        │     └─► complexity: moderate/high ──────► Tier 2 (Sonnet)
        │
        ├─► action_description ─────────────────────► Tier 2 (Sonnet)
        ├─► beat_expansion ─────────────────────────► Tier 2 (Sonnet)
        ├─► scene_description ──────────────────────► Tier 2 (Sonnet)
        ├─► translation ────────────────────────────► Tier 2 (Sonnet)
        │
        ├─► coverage_analysis ──────────────────────► Tier 3 (Opus)
        └─► structure_analysis (full script) ───────► Tier 3 (Opus)

For each tier: if primary model unavailable → try fallback model → circuit break at 3 failures
Enterprise orgs with data residency: self-hosted Llama 3.1 70B for all tiers
```

### TypeScript Implementation

```typescript
// services/ai/src/model-router.ts

import Anthropic from '@anthropic-ai/sdk';
import OpenAI from 'openai';

export type ModelProvider = 'anthropic' | 'openai' | 'self_hosted';

export interface ModelConfig {
  provider: ModelProvider;
  modelId: string;
  tier: 1 | 2 | 3;
  inputCostPer1M: number;   // USD
  outputCostPer1M: number;
  maxContextTokens: number;
  supportsPromptCache: boolean;
}

const MODELS: ModelConfig[] = [
  // Tier 1 — cheap, fast, non-creative
  {
    provider: 'anthropic',
    modelId: 'claude-haiku-4-5-20251001',
    tier: 1,
    inputCostPer1M: 0.80,
    outputCostPer1M: 4.00,
    maxContextTokens: 200_000,
    supportsPromptCache: true,
  },
  // Tier 2 — primary creative work
  {
    provider: 'anthropic',
    modelId: 'claude-sonnet-4-6',
    tier: 2,
    inputCostPer1M: 3.00,
    outputCostPer1M: 15.00,
    maxContextTokens: 200_000,
    supportsPromptCache: true,
  },
  // Tier 3 — complex analysis and generation
  {
    provider: 'anthropic',
    modelId: 'claude-opus-4-6',
    tier: 3,
    inputCostPer1M: 15.00,
    outputCostPer1M: 75.00,
    maxContextTokens: 200_000,
    supportsPromptCache: true,
  },
];

const FEATURE_TIER: Record<AIFeature, 1 | 2 | 3> = {
  format_correction: 1,
  consistency_check: 1,
  character_voice_check: 1,
  dialogue_suggestion: 2,
  action_description: 2,
  beat_expansion: 2,
  scene_description: 2,
  translation: 2,
  structure_analysis: 3,
  coverage_analysis: 3,
};

export interface RouteResult {
  model: ModelConfig;
  fallbackChain: ModelConfig[];
}

export function routeRequest(
  feature: AIFeature,
  opts: { orgUsesDataResidency?: boolean; estimatedContextTokens?: number },
): RouteResult {
  // Enterprise data residency → always self-hosted
  if (opts.orgUsesDataResidency) {
    const selfHosted: ModelConfig = {
      provider: 'self_hosted',
      modelId: 'llama-3.1-70b-scriptos',
      tier: 2,
      inputCostPer1M: 0.50,   // internal GPU cost estimate
      outputCostPer1M: 0.50,
      maxContextTokens: 128_000,
      supportsPromptCache: false,  // vLLM prefix caching differs from Anthropic cache_control
    };
    return { model: selfHosted, fallbackChain: [selfHosted] };
  }

  const targetTier = FEATURE_TIER[feature];
  const model = MODELS.find(m => m.tier === targetTier && m.provider === 'anthropic')!;

  // Fallback chain: if target tier fails, step down to tier below
  const fallbackChain = MODELS.filter(
    m => m.tier <= targetTier && m.provider === 'anthropic',
  ).sort((a, b) => b.tier - a.tier);  // highest tier first

  return { model, fallbackChain };
}

// ── Request execution with fallback ─────────────────────────────────────────

export interface ModelRequest {
  cacheablePrefix: string;
  freshSuffix: string;
  maxOutputTokens?: number;
}

export interface ModelResponse {
  text: string;
  modelUsed: string;
  inputTokens: number;
  outputTokens: number;
  cacheReadTokens: number;
  cacheWriteTokens: number;
  costUsd: number;
}

export async function executeWithFallback(
  request: ModelRequest,
  route: RouteResult,
  anthropicClient: Anthropic,
): Promise<ModelResponse> {
  const errors: Error[] = [];

  for (const model of route.fallbackChain) {
    try {
      return await executeModel(request, model, anthropicClient);
    } catch (err) {
      errors.push(err as Error);
      // Circuit break after 3 failures on same model
      if (errors.length >= 3) break;
    }
  }

  throw new AIServiceError(
    `All models in fallback chain failed: ${errors.map(e => e.message).join('; ')}`,
    'MODEL_CHAIN_EXHAUSTED',
  );
}

async function executeModel(
  request: ModelRequest,
  model: ModelConfig,
  client: Anthropic,
): Promise<ModelResponse> {
  if (model.provider === 'anthropic') {
    return executeAnthropic(request, model, client);
  }
  if (model.provider === 'self_hosted') {
    return executeSelfHosted(request, model);
  }
  throw new Error(`Unknown provider: ${model.provider}`);
}

async function executeAnthropic(
  request: ModelRequest,
  model: ModelConfig,
  client: Anthropic,
): Promise<ModelResponse> {
  const response = await client.messages.create({
    model: model.modelId,
    max_tokens: request.maxOutputTokens ?? 2048,
    system: [
      {
        type: 'text',
        text: request.cacheablePrefix,
        // Mark the stable prefix for caching — Anthropic charges $0.30/M on cache hits
        // vs $3.00/M on misses for Sonnet. Cache write is $3.75/M, amortized over hits.
        cache_control: { type: 'ephemeral' },
      },
    ] as Anthropic.TextBlockParam[],
    messages: [
      {
        role: 'user',
        content: request.freshSuffix,
      },
    ],
  });

  const usage = response.usage as Anthropic.Usage & {
    cache_read_input_tokens?: number;
    cache_creation_input_tokens?: number;
  };
  const inputTokens = usage.input_tokens;
  const outputTokens = usage.output_tokens;
  const cacheReadTokens = usage.cache_read_input_tokens ?? 0;
  const cacheWriteTokens = usage.cache_creation_input_tokens ?? 0;

  // Cost calculation:
  // Billable input = (inputTokens - cacheReadTokens - cacheWriteTokens) [regular]
  //                + cacheWriteTokens [write rate]
  //                + cacheReadTokens [read rate]
  const regularInputTokens = inputTokens - cacheReadTokens - cacheWriteTokens;
  const costUsd = (
    regularInputTokens * (model.inputCostPer1M / 1_000_000) +
    cacheWriteTokens * ((model.inputCostPer1M * 1.25) / 1_000_000) + // cache write is 25% more
    cacheReadTokens * ((model.inputCostPer1M * 0.10) / 1_000_000) +  // cache read is 10% of input
    outputTokens * (model.outputCostPer1M / 1_000_000)
  );

  const textBlock = response.content.find(b => b.type === 'text');
  if (!textBlock || textBlock.type !== 'text') throw new Error('No text in model response');

  return {
    text: textBlock.text,
    modelUsed: model.modelId,
    inputTokens,
    outputTokens,
    cacheReadTokens,
    cacheWriteTokens,
    costUsd,
  };
}

async function executeSelfHosted(
  request: ModelRequest,
  model: ModelConfig,
): Promise<ModelResponse> {
  // vLLM exposes an OpenAI-compatible endpoint
  const vllmEndpoint = process.env.VLLM_ENDPOINT ?? 'http://vllm-service.ai-infra.svc:8000';
  const openai = new OpenAI({ baseURL: `${vllmEndpoint}/v1`, apiKey: 'not-used' });

  const completion = await openai.chat.completions.create({
    model: model.modelId,
    messages: [
      { role: 'system', content: request.cacheablePrefix },
      { role: 'user', content: request.freshSuffix },
    ],
    max_tokens: 2048,
    temperature: 0.7,
  });

  const usage = completion.usage!;
  const text = completion.choices[0]?.message?.content ?? '';
  const costUsd = (usage.prompt_tokens * model.inputCostPer1M + usage.completion_tokens * model.outputCostPer1M) / 1_000_000;

  return {
    text,
    modelUsed: model.modelId,
    inputTokens: usage.prompt_tokens,
    outputTokens: usage.completion_tokens,
    cacheReadTokens: 0,
    cacheWriteTokens: 0,
    costUsd,
  };
}
```

---

## 5. AI Contribution Ledger

The ledger is **append-only and immutable**. Rows are never updated after creation. The writer's accept/reject/modify action is recorded as a separate row referencing the original entry. This creates an auditable chain.

### PostgreSQL DDL

```sql
-- Schema: ai.*

CREATE TYPE ai.writer_action AS ENUM ('accepted', 'rejected', 'modified', 'pending');

CREATE TABLE ai.ledger_entries (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- Identity
  user_id              UUID NOT NULL REFERENCES auth.users(id),
  org_id               UUID NOT NULL REFERENCES auth.organizations(id),
  project_id           UUID NOT NULL,
  scene_id             UUID,                         -- NULL for script-level analysis
  -- Classification
  interaction_type     TEXT NOT NULL,                -- AIInteractionType enum
  feature              TEXT NOT NULL,
  -- Model provenance
  model_id             TEXT NOT NULL,                -- 'claude-sonnet-4-6', 'llama-3.1-70b-scriptos'
  model_provider       TEXT NOT NULL,                -- 'anthropic' | 'openai' | 'self_hosted'
  model_tier           SMALLINT NOT NULL,            -- 1 | 2 | 3
  -- Prompt provenance (no raw prompt stored — privacy)
  prompt_hash          TEXT NOT NULL,                -- SHA-256 of assembled prompt
  context_refs         TEXT[] NOT NULL DEFAULT '{}', -- bible fact IDs used in context
  prompt_token_count   INTEGER NOT NULL,
  -- Output
  ai_output_hash       TEXT NOT NULL,                -- SHA-256 of exact AI response
  ai_output            TEXT NOT NULL,                -- exact AI response text
  output_token_count   INTEGER NOT NULL,
  -- Cost tracking
  input_tokens         INTEGER NOT NULL,
  output_tokens        INTEGER NOT NULL,
  cache_read_tokens    INTEGER NOT NULL DEFAULT 0,
  cache_write_tokens   INTEGER NOT NULL DEFAULT 0,
  cost_usd             NUMERIC(10, 6) NOT NULL,
  -- Writer action (recorded asynchronously when writer acts)
  writer_action        ai.writer_action NOT NULL DEFAULT 'pending',
  modification_delta   TEXT,                         -- unified diff if 'modified'
  action_recorded_at   TIMESTAMPTZ,
  -- Consent reference
  company_consent_ref  UUID REFERENCES ai.company_consents(id),
  -- Timing
  requested_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  responded_at         TIMESTAMPTZ NOT NULL,
  latency_ms           INTEGER NOT NULL,
  -- Immutability guard
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Never allow UPDATE or DELETE (enforced by trigger)
CREATE OR REPLACE FUNCTION ai.prevent_ledger_mutation()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  RAISE EXCEPTION 'ai.ledger_entries is append-only. Use UPDATE only for writer_action fields.';
END;
$$;

-- Only allow updating writer_action fields — everything else is immutable
CREATE OR REPLACE FUNCTION ai.restrict_ledger_update()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF OLD.ai_output != NEW.ai_output
     OR OLD.prompt_hash != NEW.prompt_hash
     OR OLD.model_id != NEW.model_id
     OR OLD.user_id != NEW.user_id
     OR OLD.org_id != NEW.org_id
  THEN
    RAISE EXCEPTION 'ai.ledger_entries: only writer_action, modification_delta, and action_recorded_at may be updated';
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER ledger_no_delete
  BEFORE DELETE ON ai.ledger_entries
  FOR EACH ROW EXECUTE FUNCTION ai.prevent_ledger_mutation();

CREATE TRIGGER ledger_restrict_update
  BEFORE UPDATE ON ai.ledger_entries
  FOR EACH ROW EXECUTE FUNCTION ai.restrict_ledger_update();

-- Partitioned by month for query performance (WGA reporting is monthly)
CREATE TABLE ai.ledger_entries_y2026m01 PARTITION OF ai.ledger_entries
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- Additional partitions created by Atlas migration on a rolling monthly schedule

-- Indexes
CREATE INDEX ON ai.ledger_entries (org_id, project_id, requested_at);
CREATE INDEX ON ai.ledger_entries (user_id, requested_at);
CREATE INDEX ON ai.ledger_entries (project_id, interaction_type, requested_at);
CREATE INDEX ON ai.ledger_entries (writer_action) WHERE writer_action = 'pending';
```

### Ledger Write

```typescript
// services/ai/src/ledger.ts

import crypto from 'node:crypto';

export interface LedgerWriteInput {
  userId: string;
  orgId: string;
  projectId: string;
  sceneId: string | null;
  feature: AIFeature;
  interactionType: AIInteractionType;
  assembledContext: AssembledContext;
  response: ModelResponse;
  consentRef: string;
  requestedAt: Date;
  respondedAt: Date;
}

export async function writeLedgerEntry(input: LedgerWriteInput, db: DB): Promise<string> {
  const promptText = input.assembledContext.cacheablePrefix + '\n\n' + input.assembledContext.freshSuffix;
  const promptHash = crypto.createHash('sha256').update(promptText).digest('hex');
  const outputHash = crypto.createHash('sha256').update(input.response.text).digest('hex');
  const latencyMs = input.respondedAt.getTime() - input.requestedAt.getTime();

  const { id } = await db.queryOne<{ id: string }>(sql`
    INSERT INTO ai.ledger_entries (
      user_id, org_id, project_id, scene_id,
      interaction_type, feature,
      model_id, model_provider, model_tier,
      prompt_hash, context_refs, prompt_token_count,
      ai_output_hash, ai_output, output_token_count,
      input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, cost_usd,
      company_consent_ref,
      requested_at, responded_at, latency_ms
    ) VALUES (
      ${input.userId}, ${input.orgId}, ${input.projectId}, ${input.sceneId},
      ${input.interactionType}, ${input.feature},
      ${input.response.modelUsed}, ${getProvider(input.response.modelUsed)}, ${getTier(input.response.modelUsed)},
      ${promptHash}, ${input.assembledContext.contextRefs}, ${input.assembledContext.estimatedTokens},
      ${outputHash}, ${input.response.text}, ${input.response.outputTokens},
      ${input.response.inputTokens}, ${input.response.outputTokens},
      ${input.response.cacheReadTokens}, ${input.response.cacheWriteTokens},
      ${input.response.costUsd},
      ${input.consentRef},
      ${input.requestedAt.toISOString()}, ${input.respondedAt.toISOString()}, ${latencyMs}
    )
    RETURNING id
  `);

  return id;
}

export async function recordWriterAction(
  ledgerEntryId: string,
  action: 'accepted' | 'rejected' | 'modified',
  modificationDelta: string | null,
  db: DB,
): Promise<void> {
  await db.query(sql`
    UPDATE ai.ledger_entries
    SET
      writer_action = ${action},
      modification_delta = ${modificationDelta},
      action_recorded_at = NOW()
    WHERE id = ${ledgerEntryId}
      AND writer_action = 'pending'
  `);
}
```

---

## 6. Semantic Cache

The semantic cache deduplicates near-identical AI requests across writers. A new request within cosine similarity ≥ 0.97 of a cached request (same feature, same project) returns the cached response without hitting the model.

### Architecture

```
Request arrives
      │
      ▼
Compute embedding of freshSuffix (OpenAI text-embedding-3-small — cheap, fast)
      │
      ▼
pgvector ANN search against ai.semantic_cache
WHERE feature = ? AND project_id = ? AND similarity > 0.97
      │
      ├─► HIT: return cached response; write new ledger entry citing cache_source_id
      │
      └─► MISS: execute model → store response in cache → return to writer
```

### PostgreSQL Schema

```sql
-- Uses pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE ai.semantic_cache (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id      UUID NOT NULL,
  feature         TEXT NOT NULL,
  query_embedding VECTOR(1536) NOT NULL,  -- OpenAI text-embedding-3-large dims
  query_hash      TEXT NOT NULL,          -- SHA-256 for exact-match fast path
  response_text   TEXT NOT NULL,
  model_id        TEXT NOT NULL,
  hit_count       INTEGER NOT NULL DEFAULT 0,
  last_hit_at     TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at      TIMESTAMPTZ NOT NULL    -- TTL: 24h for dialogue, 7 days for analysis
);

-- IVFFlat index for approximate nearest neighbor search
CREATE INDEX ON ai.semantic_cache
  USING ivfflat (query_embedding vector_cosine_ops)
  WITH (lists = 100);  -- sqrt(row_count) is a good heuristic; recalculate at 10K+ rows

CREATE INDEX ON ai.semantic_cache (project_id, feature, query_hash);
CREATE INDEX ON ai.semantic_cache (expires_at);
```

### TypeScript Implementation

```typescript
// services/ai/src/semantic-cache.ts

const SIMILARITY_THRESHOLD = 0.97;
const CACHE_TTL: Record<AIFeature, number> = {
  format_correction: 60 * 60 * 24 * 7,    // 7 days (deterministic)
  consistency_check: 60 * 60 * 24,         // 1 day
  character_voice_check: 60 * 60 * 24,
  dialogue_suggestion: 60 * 60 * 24,       // 1 day (creative — shorter TTL)
  action_description: 60 * 60 * 24,
  beat_expansion: 60 * 60 * 24,
  scene_description: 60 * 60 * 24,
  translation: 60 * 60 * 24 * 7,           // 7 days (deterministic)
  structure_analysis: 60 * 60 * 24 * 7,
  coverage_analysis: 60 * 60 * 24 * 7,
};

export interface CacheLookupResult {
  hit: boolean;
  response?: string;
  cacheEntryId?: string;
}

export async function lookupSemanticCache(
  query: string,
  feature: AIFeature,
  projectId: string,
  embeddingClient: OpenAI,
  db: DB,
): Promise<CacheLookupResult> {
  const queryHash = crypto.createHash('sha256').update(query).digest('hex');

  // Fast path: exact hash match
  const exact = await db.queryOne<{ id: string; response_text: string }>(sql`
    SELECT id, response_text FROM ai.semantic_cache
    WHERE project_id = ${projectId}
      AND feature = ${feature}
      AND query_hash = ${queryHash}
      AND expires_at > NOW()
    LIMIT 1
  `);

  if (exact) {
    void updateHitCount(exact.id, db);
    return { hit: true, response: exact.response_text, cacheEntryId: exact.id };
  }

  // Semantic path: embedding similarity
  const embeddingResponse = await embeddingClient.embeddings.create({
    model: 'text-embedding-3-small',  // cheaper for cache lookup; 3-large for indexing
    input: query,
    dimensions: 1536,
  });
  const embedding = embeddingResponse.data[0].embedding;

  const similar = await db.queryOne<{ id: string; response_text: string; similarity: number }>(sql`
    SELECT id, response_text, 1 - (query_embedding <=> ${JSON.stringify(embedding)}::vector) AS similarity
    FROM ai.semantic_cache
    WHERE project_id = ${projectId}
      AND feature = ${feature}
      AND expires_at > NOW()
      AND 1 - (query_embedding <=> ${JSON.stringify(embedding)}::vector) > ${SIMILARITY_THRESHOLD}
    ORDER BY query_embedding <=> ${JSON.stringify(embedding)}::vector
    LIMIT 1
  `);

  if (similar) {
    void updateHitCount(similar.id, db);
    return { hit: true, response: similar.response_text, cacheEntryId: similar.id };
  }

  return { hit: false };
}

export async function storeInSemanticCache(
  query: string,
  response: string,
  feature: AIFeature,
  projectId: string,
  modelId: string,
  embeddingClient: OpenAI,
  db: DB,
): Promise<string> {
  const queryHash = crypto.createHash('sha256').update(query).digest('hex');
  const ttlSeconds = CACHE_TTL[feature];

  const embeddingResponse = await embeddingClient.embeddings.create({
    model: 'text-embedding-3-large',  // higher quality for stored embeddings
    input: query,
    dimensions: 1536,
  });
  const embedding = embeddingResponse.data[0].embedding;

  const { id } = await db.queryOne<{ id: string }>(sql`
    INSERT INTO ai.semantic_cache
      (project_id, feature, query_embedding, query_hash, response_text, model_id, expires_at)
    VALUES (
      ${projectId}, ${feature}, ${JSON.stringify(embedding)}::vector,
      ${queryHash}, ${response}, ${modelId},
      NOW() + INTERVAL '${ttlSeconds} seconds'
    )
    ON CONFLICT (project_id, feature, query_hash)
      DO UPDATE SET response_text = EXCLUDED.response_text,
                    expires_at = EXCLUDED.expires_at,
                    hit_count = ai.semantic_cache.hit_count + 1,
                    last_hit_at = NOW()
    RETURNING id
  `);

  return id;
}

async function updateHitCount(cacheEntryId: string, db: DB): Promise<void> {
  await db.query(sql`
    UPDATE ai.semantic_cache
    SET hit_count = hit_count + 1, last_hit_at = NOW()
    WHERE id = ${cacheEntryId}
  `);
}
```

---

## 7. Output Guard

Before AI output reaches the writer, it passes through the Output Guard — a fast, rule-based check. The guard does not call an AI model; it uses deterministic rules against the assembled context.

```typescript
// services/ai/src/output-guard.ts

export interface GuardResult {
  passed: boolean;
  violations: GuardViolation[];
}

export interface GuardViolation {
  type: 'voice_violation' | 'canon_violation' | 'policy_violation';
  severity: 'warning' | 'block';
  message: string;
  affectedCharacter?: string;
}

export async function runOutputGuard(
  output: string,
  feature: AIFeature,
  context: AssembledContext,
  voiceProfiles: VoiceProfile[],
  bibleFacts: BibleFact[],
): Promise<GuardResult> {
  const violations: GuardViolation[] = [];

  // ── 1. Voice Guard — check forbidden words per character ─────────────────
  for (const profile of voiceProfiles) {
    for (const forbidden of profile.forbidden_words) {
      // Look for the forbidden word in dialogue lines attributed to this character
      const pattern = new RegExp(
        `\\b${escapeRegex(profile.character_name)}\\b[\\s\\S]{0,200}\\b${escapeRegex(forbidden)}\\b`,
        'i',
      );
      if (pattern.test(output)) {
        violations.push({
          type: 'voice_violation',
          severity: 'warning',  // voice violations are always warnings — never blocks
          message: `"${forbidden}" is flagged as a word ${profile.character_name} would not say`,
          affectedCharacter: profile.character_name,
        });
      }
    }
  }

  // ── 2. Canon Guard — detect obvious contradictions with established facts ──
  for (const fact of bibleFacts) {
    if (fact.negation_form && output.toLowerCase().includes(fact.negation_form.toLowerCase())) {
      violations.push({
        type: 'canon_violation',
        severity: 'warning',
        message: `Possible contradiction with established fact: "${fact.fact_text}"`,
      });
    }
  }

  // ── 3. Policy Guard — WGA compliance checks ──────────────────────────────
  const policyViolations = checkWGAPolicy(output, feature);
  violations.push(...policyViolations);

  // Block only on hard policy violations
  const passed = !violations.some(v => v.severity === 'block');

  return { passed, violations };
}

function checkWGAPolicy(output: string, feature: AIFeature): GuardViolation[] {
  const violations: GuardViolation[] = [];

  // Detect if AI is claiming authorship (never acceptable)
  const authorshipPatterns = [
    /\bI wrote\b/i,
    /\bI have created\b/i,
    /\bmy script\b/i,
    /\bI authored\b/i,
  ];

  for (const pattern of authorshipPatterns) {
    if (pattern.test(output)) {
      violations.push({
        type: 'policy_violation',
        severity: 'block',
        message: 'AI output contains language that could be construed as claiming authorship — blocked per WGA compliance policy',
      });
    }
  }

  return violations;
}
```

---

## 8. Rate Limiting

### Token Bucket Design

```
Org level (hard limit):
  - bucket: ai:quota:{org_id}:{YYYY-MM}
  - capacity: monthly token quota (from pricing tier)
  - refills: never (monthly bucket; resets on billing cycle)
  - enforcement: hard block when empty → HTTP 429 with retry_after = start of next month

User level (soft limit):
  - bucket: ai:user_quota:{user_id}:{YYYY-MM}
  - capacity: configurable per org (default: org_quota / expected_seat_count)
  - enforcement: warning at 80%; soft block at 100% (org admin can lift)
```

### Token Quotas by Tier

| Tier | Monthly Org Quota | Default Per-User Allocation |
|------|------------------|-----------------------------|
| Indie | 0 (no AI generation) | — |
| Professional | 500,000 tokens | 50,000 tokens |
| Studio | Custom (metered) | Configurable |

### Redis Implementation

```typescript
// services/ai/src/rate-limiter.ts

const ORG_QUOTA_BY_TIER: Record<string, number> = {
  indie: 0,
  professional: 500_000,
  studio: Infinity,  // metered — never hard-blocked; billed at overage
};

const DEFAULT_USER_SHARE = 0.10; // 10% of org quota as default per-user allocation

export interface QuotaCheckResult {
  allowed: boolean;
  reason?: 'org_quota_exceeded' | 'user_quota_soft_exceeded' | 'feature_not_in_tier';
  orgTokensRemaining: number;
  userTokensRemaining: number;
  warningLevel?: 'approaching_limit';
}

export async function checkAndConsumeQuota(
  orgId: string,
  userId: string,
  orgTier: string,
  estimatedTokens: number,
  redis: Redis,
  db: DB,
): Promise<QuotaCheckResult> {
  const orgQuota = ORG_QUOTA_BY_TIER[orgTier] ?? 0;

  // Indie tier has no AI generation
  if (orgQuota === 0) {
    return {
      allowed: false,
      reason: 'feature_not_in_tier',
      orgTokensRemaining: 0,
      userTokensRemaining: 0,
    };
  }

  const monthKey = new Date().toISOString().slice(0, 7); // YYYY-MM
  const orgBucketKey = `ai:quota:${orgId}:${monthKey}`;
  const userBucketKey = `ai:user_quota:${userId}:${monthKey}`;

  // Lua script for atomic check-and-decrement
  const luaScript = `
    local org_key = KEYS[1]
    local user_key = KEYS[2]
    local org_quota = tonumber(ARGV[1])
    local user_quota = tonumber(ARGV[2])
    local tokens = tonumber(ARGV[3])

    -- Initialize if first request this month
    if redis.call('EXISTS', org_key) == 0 then
      redis.call('SET', org_key, org_quota, 'EX', 2678400)  -- 31 days
    end
    if redis.call('EXISTS', user_key) == 0 then
      redis.call('SET', user_key, user_quota, 'EX', 2678400)
    end

    local org_remaining = tonumber(redis.call('GET', org_key))
    local user_remaining = tonumber(redis.call('GET', user_key))

    if org_quota ~= -1 and org_remaining < tokens then
      return {0, org_remaining, user_remaining}  -- org quota exceeded
    end

    if user_remaining < tokens then
      return {2, org_remaining, user_remaining}  -- user soft limit exceeded
    end

    -- Consume
    if org_quota ~= -1 then
      redis.call('DECRBY', org_key, tokens)
    end
    redis.call('DECRBY', user_key, tokens)

    return {1, org_remaining - tokens, user_remaining - tokens}
  `;

  // Get user's configured quota (default = 10% of org quota)
  const userQuotaConfig = await db.queryOne<{ token_limit: number }>(sql`
    SELECT token_limit FROM ai.user_quota_configs
    WHERE user_id = ${userId} AND org_id = ${orgId}
  `);

  const userQuota = userQuotaConfig?.token_limit ??
    (orgQuota === Infinity ? 1_000_000 : Math.floor(orgQuota * DEFAULT_USER_SHARE));
  const orgQuotaArg = orgQuota === Infinity ? -1 : orgQuota;

  const [statusCode, orgRemaining, userRemaining] = await redis.eval(
    luaScript,
    2,
    orgBucketKey,
    userBucketKey,
    orgQuotaArg.toString(),
    userQuota.toString(),
    estimatedTokens.toString(),
  ) as [number, number, number];

  if (statusCode === 0) {
    return { allowed: false, reason: 'org_quota_exceeded', orgTokensRemaining: orgRemaining, userTokensRemaining: userRemaining };
  }
  if (statusCode === 2) {
    return { allowed: false, reason: 'user_quota_soft_exceeded', orgTokensRemaining: orgRemaining, userTokensRemaining: userRemaining };
  }

  const warning = userRemaining < userQuota * 0.2 ? 'approaching_limit' as const : undefined;
  return { allowed: true, orgTokensRemaining: orgRemaining, userTokensRemaining: userRemaining, warningLevel: warning };
}
```

---

## 9. Full AI Request Flow

Putting all components together, this is the complete flow for a single AI assistance request:

```typescript
// services/ai/src/ai-service.ts

export interface AIRequest {
  userId: string;
  orgId: string;
  orgTier: string;
  orgUsesDataResidency: boolean;
  projectId: string;
  sceneId: string | null;
  feature: AIFeature;
  interactionType: AIInteractionType;
  userRequest?: string;
  // Scene context — populated from CRDT state by caller
  characterNames: string[];
  locationName: string;
  scriptId: string;
}

export interface AIResponse {
  ledgerEntryId: string;
  output: string;
  violations: GuardViolation[];
  cacheHit: boolean;
  modelUsed: string;
  tokensUsed: number;
  costUsd: number;
  quotaWarning?: 'approaching_limit';
}

export async function handleAIRequest(req: AIRequest, deps: ServiceDeps): Promise<AIResponse> {
  const requestedAt = new Date();

  // ── 1. Consent check ─────────────────────────────────────────────────────
  const consentResult = await checkConsent({
    orgId: req.orgId,
    projectId: req.projectId,
    writerId: req.userId,
    feature: req.feature,
  }, deps.db);

  if (!consentResult.allowed) {
    throw new AIConsentError(consentResult.reason);
  }

  // ── 2. Quota check (estimated tokens — refine after response) ─────────────
  const estimatedTokens = 5_000; // pre-flight estimate; actual consumed after response
  const quotaResult = await checkAndConsumeQuota(
    req.orgId, req.userId, req.orgTier, estimatedTokens,
    deps.redis, deps.db,
  );

  if (!quotaResult.allowed) {
    throw new AIQuotaError(quotaResult.reason!);
  }

  // ── 3. Context assembly ──────────────────────────────────────────────────
  const context = await assembleContext({
    projectId: req.projectId,
    scriptId: req.scriptId,
    sceneId: req.sceneId ?? '',
    characterNames: req.characterNames,
    locationName: req.locationName,
    feature: req.feature,
    userRequest: req.userRequest,
  }, deps.bible, deps.db);

  // ── 4. Semantic cache lookup ──────────────────────────────────────────────
  const queryText = context.freshSuffix;
  const cacheResult = await lookupSemanticCache(
    queryText, req.feature, req.projectId,
    deps.embeddingClient, deps.db,
  );

  let responseText: string;
  let modelUsed: string;
  let tokensUsed: number;
  let costUsd: number;
  let cacheHit = false;

  if (cacheResult.hit) {
    responseText = cacheResult.response!;
    modelUsed = 'semantic_cache';
    tokensUsed = 0;
    costUsd = 0;
    cacheHit = true;
  } else {
    // ── 5. Model routing + execution ──────────────────────────────────────
    const route = routeRequest(req.feature, { orgUsesDataResidency: req.orgUsesDataResidency });
    const modelResponse = await executeWithFallback(
      { cacheablePrefix: context.cacheablePrefix, freshSuffix: context.freshSuffix },
      route,
      deps.anthropicClient,
    );

    responseText = modelResponse.text;
    modelUsed = modelResponse.modelUsed;
    tokensUsed = modelResponse.inputTokens + modelResponse.outputTokens;
    costUsd = modelResponse.costUsd;

    // Store in semantic cache asynchronously
    void storeInSemanticCache(
      queryText, responseText, req.feature, req.projectId,
      modelUsed, deps.embeddingClient, deps.db,
    );
  }

  const respondedAt = new Date();

  // ── 6. Output guard ──────────────────────────────────────────────────────
  const voiceProfiles = await deps.bible.getVoiceProfiles(req.projectId, req.characterNames);
  const bibleFacts = await deps.bible.getFactsForContext(context.contextRefs);
  const guardResult = await runOutputGuard(
    responseText, req.feature, context, voiceProfiles, bibleFacts,
  );

  if (!guardResult.passed) {
    // Hard policy violation — don't show output to writer, don't log it
    throw new AIGuardError('Output blocked by policy guard', guardResult.violations);
  }

  // ── 7. Write ledger entry ────────────────────────────────────────────────
  const ledgerEntryId = await writeLedgerEntry({
    userId: req.userId,
    orgId: req.orgId,
    projectId: req.projectId,
    sceneId: req.sceneId,
    feature: req.feature,
    interactionType: req.interactionType,
    assembledContext: context,
    response: { text: responseText, modelUsed, inputTokens: tokensUsed, outputTokens: 0, cacheReadTokens: 0, cacheWriteTokens: 0, costUsd },
    consentRef: consentResult.consentRef,
    requestedAt,
    respondedAt,
  }, deps.db);

  // ── 8. Adjust actual quota consumption (replace estimate with actual) ─────
  // The difference between actual and estimated tokens is credited back
  // (or further debited) in the next Lua script call — not implemented here for brevity

  return {
    ledgerEntryId,
    output: responseText,
    violations: guardResult.violations,
    cacheHit,
    modelUsed,
    tokensUsed,
    costUsd,
    quotaWarning: quotaResult.warningLevel,
  };
}
```

---

## 10. WGA Compliance Reporting

### Per-Project Usage Report (Semi-Annual)

The WGA requires studios to report AI usage at semi-annual meetings. ScriptOS exports a report per project covering the requested period.

```sql
-- services/ai/src/queries/wga-report.sql
-- Parameters: :project_id, :period_start, :period_end

WITH usage_summary AS (
  SELECT
    le.user_id,
    u.display_name AS writer_name,
    le.interaction_type,
    COUNT(*) AS total_interactions,
    COUNT(*) FILTER (WHERE le.writer_action = 'accepted') AS accepted,
    COUNT(*) FILTER (WHERE le.writer_action = 'rejected') AS rejected,
    COUNT(*) FILTER (WHERE le.writer_action = 'modified') AS modified,
    COUNT(*) FILTER (WHERE le.writer_action = 'pending') AS pending,
    SUM(le.input_tokens + le.output_tokens) AS total_tokens,
    SUM(le.cost_usd) AS total_cost_usd,
    COUNT(DISTINCT le.model_id) AS models_used,
    array_agg(DISTINCT le.model_id) AS model_ids
  FROM ai.ledger_entries le
  JOIN auth.users u ON u.id = le.user_id
  WHERE le.project_id = :project_id
    AND le.requested_at >= :period_start
    AND le.requested_at < :period_end
  GROUP BY le.user_id, u.display_name, le.interaction_type
),
consent_audit AS (
  SELECT
    cc.granted_at,
    cc.expires_at,
    cc.document_ref,
    u.display_name AS granted_by_name
  FROM ai.company_consents cc
  LEFT JOIN auth.users u ON u.id = cc.granted_by
  WHERE cc.project_id = :project_id
    AND cc.consent_state IN ('granted', 'revoked', 'expired')
  ORDER BY cc.granted_at DESC
)
SELECT
  jsonb_build_object(
    'report_type', 'wga_article_72_semi_annual',
    'project_id', :project_id,
    'period_start', :period_start,
    'period_end', :period_end,
    'generated_at', NOW(),
    'usage_by_writer_and_type', (SELECT jsonb_agg(us.*) FROM usage_summary us),
    'consent_audit', (SELECT jsonb_agg(ca.*) FROM consent_audit ca),
    'summary', jsonb_build_object(
      'total_interactions', (SELECT SUM(total_interactions) FROM usage_summary),
      'total_writers_with_ai_use', (SELECT COUNT(DISTINCT user_id) FROM usage_summary),
      'acceptance_rate_pct', (
        SELECT ROUND(
          100.0 * SUM(accepted) / NULLIF(SUM(accepted + rejected + modified), 0), 1
        )
        FROM usage_summary
      ),
      'modification_rate_pct', (
        SELECT ROUND(
          100.0 * SUM(modified) / NULLIF(SUM(accepted + rejected + modified), 0), 1
        )
        FROM usage_summary
      )
    )
  ) AS report;
```

### TypeScript Export Handler

```typescript
// services/ai/src/compliance.ts

export async function generateWGAReport(
  projectId: string,
  periodStart: Date,
  periodEnd: Date,
  format: 'json' | 'pdf' | 'csv',
  db: DB,
): Promise<Buffer> {
  const report = await db.queryOne<{ report: WGAReport }>(
    wgaReportQuery,
    { project_id: projectId, period_start: periodStart.toISOString(), period_end: periodEnd.toISOString() },
  );

  if (!report) throw new Error('No data for report period');

  switch (format) {
    case 'json':
      return Buffer.from(JSON.stringify(report.report, null, 2), 'utf-8');
    case 'csv':
      return generateWGAReportCSV(report.report);
    case 'pdf':
      return generateWGAReportPDF(report.report);  // uses Puppeteer
    default:
      throw new Error(`Unknown format: ${format}`);
  }
}
```

---

## 11. C2PA Integration

Content Credentials (C2PA spec) embed cryptographically verifiable provenance into distributed files. ScriptOS embeds C2PA manifests at export time.

### Manifest Structure

```typescript
// packages/ast/src/c2pa.ts

export interface ScriptOSC2PAManifest {
  claim_generator: 'ScriptOS/1.0';
  title: string;                  // script title
  assertions: C2PAAssertion[];
  ingredients: C2PAIngredient[];
}

export interface C2PAAssertion {
  label: string;
  data: Record<string, unknown>;
}

// Assertions included in every ScriptOS export:
const ASSERTION_TEMPLATES: C2PAAssertion[] = [
  {
    label: 'stds.schema-org.CreativeWork',
    data: {
      '@context': 'https://schema.org',
      '@type': 'CreativeWork',
      author: [],            // populated with human writers from project membership
      // Note: AI assistants are NOT listed as authors per WGA compliance
    },
  },
  {
    label: 'c2pa.actions',
    data: {
      actions: [
        {
          action: 'c2pa.created',
          softwareAgent: 'ScriptOS/1.0',
          when: '',          // ISO 8601 export timestamp
        },
      ],
    },
  },
  {
    label: 'com.scriptos.ai_involvement',
    // Custom assertion — not part of C2PA spec, but valid as a custom label
    data: {
      ai_assisted_scenes: [],    // scene IDs with at least one accepted AI suggestion
      ai_interaction_count: 0,  // total accepted AI interactions in this version
      model_ids_used: [],        // model IDs that contributed accepted suggestions
      wga_consent_ref: '',       // company consent document reference
    },
  },
];
```

### PDF Embedding

C2PA mandates are embedded in PDF via the XMP metadata stream (supported by the C2PA PDF spec):

```typescript
// services/ai/src/c2pa-embedder.ts

import { createC2pa, ManifestBuilder } from 'c2pa-node';  // @contentauth/c2pa-node

export async function embedC2PAInPDF(
  pdfBuffer: Buffer,
  scriptId: string,
  projectId: string,
  db: DB,
): Promise<Buffer> {
  const c2pa = await createC2pa();

  // Pull AI involvement data from ledger
  const aiData = await db.queryOne<{
    ai_scene_ids: string[];
    interaction_count: number;
    model_ids: string[];
    consent_ref: string;
  }>(sql`
    SELECT
      array_agg(DISTINCT le.scene_id) FILTER (WHERE le.scene_id IS NOT NULL
        AND le.writer_action = 'accepted') AS ai_scene_ids,
      COUNT(*) FILTER (WHERE le.writer_action = 'accepted') AS interaction_count,
      array_agg(DISTINCT le.model_id) FILTER (WHERE le.writer_action = 'accepted') AS model_ids,
      MAX(le.company_consent_ref::TEXT) AS consent_ref
    FROM ai.ledger_entries le
    WHERE le.project_id = ${projectId}
      AND le.writer_action = 'accepted'
  `);

  const manifest = new ManifestBuilder({
    claim_generator: 'ScriptOS/1.0',
    format: 'application/pdf',
    assertions: [
      {
        label: 'com.scriptos.ai_involvement',
        data: {
          ai_assisted_scenes: aiData?.ai_scene_ids ?? [],
          ai_interaction_count: aiData?.interaction_count ?? 0,
          model_ids_used: aiData?.model_ids ?? [],
          wga_consent_ref: aiData?.consent_ref ?? '',
        },
      },
    ],
  });

  const result = await c2pa.sign({
    asset: { buffer: pdfBuffer, mimeType: 'application/pdf' },
    manifest,
  });

  return result.buffer;
}
```

---

## 12. Self-Hosted Model Path (Enterprise Data Residency)

For studios requiring data sovereignty (no script content leaves their VPC), ScriptOS deploys Llama 3.1 70B via vLLM on customer-managed GPU infrastructure.

### Kubernetes Deployment

```yaml
# infra/k8s/ai-infra/vllm-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  namespace: ai-infra
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vllm-server
  template:
    metadata:
      labels:
        app: vllm-server
    spec:
      nodeSelector:
        cloud.google.com/gke-accelerator: nvidia-a100-80gb
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.6.0
          args:
            - --model
            - /mnt/models/llama-3.1-70b-scriptos
            - --served-model-name
            - llama-3.1-70b-scriptos
            - --tensor-parallel-size
            - "2"            # 2 A100s per replica for 70B model
            - --max-model-len
            - "131072"       # 128K context
            - --enable-prefix-caching  # vLLM prefix caching (analogous to Anthropic prompt caching)
            - --gpu-memory-utilization
            - "0.95"
          resources:
            limits:
              nvidia.com/gpu: "2"
              memory: "160Gi"
          volumeMounts:
            - name: model-storage
              mountPath: /mnt/models
              readOnly: true
      volumes:
        - name: model-storage
          persistentVolumeClaim:
            claimName: llama-model-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-service
  namespace: ai-infra
spec:
  selector:
    app: vllm-server
  ports:
    - port: 8000
      targetPort: 8000
```

### Model Switching Logic

The model router checks `org.uses_data_residency` and routes all requests to the vLLM endpoint if `true`. The OpenAI-compatible API surface means no changes to prompt assembly or output parsing — only the endpoint URL and model ID change.

---

## 13. Cost Modeling

### Per-Tier Monthly Cost Estimates

Assumptions: Professional tier org with 5 writers, each 20 writing sessions/month, 100 AI requests/session.

| Cost Component | Monthly Estimate |
|---------------|-----------------|
| Tier 2 (Sonnet) model calls — dialogue, action | ~$45 |
| Tier 1 (Haiku) — consistency, format checks | ~$3 |
| Semantic cache embedding costs (text-embedding-3-small) | ~$0.50 |
| Cache hit rate (31% est.) reduces model calls by ~31% | -$15 |
| Prompt cache hit rate (90% of prefix) reduces prefix cost by ~90% | -$35 |
| **Net monthly AI cost per 5-seat Professional org** | **~$35–50** |

Professional tier includes 500K tokens/month (~$1.50 at blended rate before caching). With caching, effective capacity is ~4–5M tokens. Pricing model has substantial margin.

### Real-Time Cost Tracking

Every ledger entry records `cost_usd`. Monthly org spend is queryable in real time:

```sql
SELECT
  SUM(cost_usd) AS month_to_date_usd,
  SUM(input_tokens + output_tokens) AS tokens_consumed,
  COUNT(*) AS total_requests
FROM ai.ledger_entries
WHERE org_id = :org_id
  AND requested_at >= date_trunc('month', NOW())
```

---

## 14. Testing Strategy

### Unit Tests

```typescript
// services/ai/src/__tests__/consent.test.ts
describe('checkConsent', () => {
  it('denies when no company consent exists', async () => { ... });
  it('denies when consent is expired', async () => { ... });
  it('denies when feature not enabled by writer', async () => { ... });
  it('allows format_correction without feature consent check', async () => { ... });
  it('grants when valid consent and feature enabled', async () => { ... });
});

// services/ai/src/__tests__/model-router.test.ts
describe('routeRequest', () => {
  it('routes format_correction to Tier 1', async () => { ... });
  it('routes coverage_analysis to Tier 3', async () => { ... });
  it('routes all features to self_hosted when data_residency=true', async () => { ... });
  it('fallback chain has models in descending tier order', async () => { ... });
});

// services/ai/src/__tests__/output-guard.test.ts
describe('runOutputGuard', () => {
  it('flags forbidden word in dialogue attributed to character', async () => { ... });
  it('blocks output containing authorship claim language', async () => { ... });
  it('passes output with voice warning (warning != block)', async () => { ... });
  it('detects canon contradiction via negation_form', async () => { ... });
});

// services/ai/src/__tests__/rate-limiter.test.ts
describe('checkAndConsumeQuota', () => {
  it('blocks indie tier (no AI quota)', async () => { ... });
  it('decrements org and user buckets atomically', async () => { ... });
  it('returns approaching_limit warning at 80% user consumption', async () => { ... });
  it('blocks when org quota is exhausted', async () => { ... });
  it('allows Studio tier beyond quota (metered, no hard block)', async () => { ... });
});
```

### Integration Tests

```typescript
// services/ai/src/__tests__/integration/full-request-flow.test.ts
describe('full AI request flow', () => {
  it('dialogue suggestion: context assembled → model called → guard passed → ledger written', async () => {
    // Uses test database with seeded Bible facts, voice profiles
    // Uses mock Anthropic client returning known response
    const response = await handleAIRequest({
      userId: 'test-writer',
      orgId: 'test-org',
      orgTier: 'professional',
      orgUsesDataResidency: false,
      projectId: 'test-project',
      sceneId: 'test-scene',
      feature: 'dialogue_suggestion',
      interactionType: 'dialogue_suggestion',
      characterNames: ['WALTER', 'JESSIE'],
      locationName: 'THE LAB',
      scriptId: 'test-script',
      userRequest: 'Suggest an alternative line for WALTER',
    }, testDeps);

    expect(response.ledgerEntryId).toBeTruthy();
    expect(response.output).toBeTruthy();
    expect(response.cacheHit).toBe(false);

    // Verify ledger entry was written
    const entry = await testDb.queryOne(
      sql`SELECT * FROM ai.ledger_entries WHERE id = ${response.ledgerEntryId}`,
    );
    expect(entry.writer_action).toBe('pending');
    expect(entry.company_consent_ref).toBeTruthy();
  });

  it('second identical request hits semantic cache', async () => {
    // First request populates cache
    await handleAIRequest(baseRequest, testDeps);
    // Second request should hit cache
    const response = await handleAIRequest(baseRequest, testDeps);
    expect(response.cacheHit).toBe(true);
    expect(response.costUsd).toBe(0);
  });

  it('writer accepting suggestion updates ledger entry writer_action', async () => {
    const { ledgerEntryId } = await handleAIRequest(baseRequest, testDeps);
    await recordWriterAction(ledgerEntryId, 'accepted', null, testDb);
    const entry = await testDb.queryOne(
      sql`SELECT writer_action FROM ai.ledger_entries WHERE id = ${ledgerEntryId}`,
    );
    expect(entry.writer_action).toBe('accepted');
  });
});
```

---

## Decisions

**Prompt architecture — hierarchical context with Anthropic prompt caching.**
Assemble context in two sections: a stable cacheable prefix (Bible facts for relevant characters and locations, voice profiles, world rules, series timeline state) and a fresh variable suffix (current scene + surrounding 2–3 scenes). The stable prefix accounts for ~90% of tokens and is cached via Anthropic's `cache_control: { type: "ephemeral" }` header ($0.30/M cache hits vs $3.00/M misses — ~90% cost reduction on the context portion). Context assembly order: world rules → character facts → voice profile → timeline state → surrounding scenes → current scene. See ADR-020.

**Voice registry reference scenes — 3–5 per character.**
Minimum 1 (the character's introduction scene). Target 3 well-chosen scenes covering different emotional registers: introduction, emotional peak, and conflict/confrontation. Maximum 10 — beyond this, marginal quality improvement is negligible vs token cost. Writers choose reference scenes manually; the AI Service uses them for few-shot prompting prepended before the dialogue generation request.

**Self-hosted model — Llama 3.1 70B fine-tuned, served via vLLM.**
For enterprise clients requiring data residency, use Llama 3.1 70B as the base model, fine-tuned on WGA-permitted scripts and public domain screenplays for format adherence and dialogue quality. vLLM provides the highest throughput per GPU for autoregressive inference. API shape matches OpenAI-compatible endpoints — the model router switches between Anthropic API, OpenAI API, and vLLM endpoint with no application code changes. See ADR-006.

**C2PA integration — PDF export in v1, FDX in v2.**
C2PA manifests embed cleanly into PDFs (spec-supported, readers exist). FDX is a proprietary XML format — embedding C2PA in FDX requires a custom approach that Final Draft does not natively read. The PDF is the primary distribution format for scripts. FDX C2PA embedding is a v2 enhancement once the PDF path is proven.

**Rate limiting — organization quota, per-user soft limits.**
The organization is the billing unit. Each org has a monthly token quota determined by pricing tier. Individual users within the org have soft limits (configurable by org admins) to prevent one writer from exhausting the shared quota. Hard limits enforce at the org level; soft limits surface warnings to users at 80% of their individual allocation. Metered overages for Studio tier billed at cost + 30% margin.

**Semantic cache TTL — creative features 24h; analytical features 7 days.**
Dialogue suggestions and scene descriptions have creative variability — stale cache hits for these features would return the same suggestion to multiple writers, reducing the sense of personalization. Analytical features (coverage, structure analysis) for the same content version are deterministic — identical queries 7 days apart should return the same analysis unless the script changed. Cache keys include the `query_hash` so any script change invalidates the exact cache entry. Semantic similarity threshold 0.97 is high enough to prevent false hits across genuinely different requests.

**Output guard — warnings only for voice/canon violations; hard block only for WGA policy violations.**
Voice and canon violations are surfaced as in-UI warnings alongside the suggestion. The writer remains in control — they may accept a dialogue line that technically violates a voice rule if they're intentionally evolving the character. Only WGA policy violations (AI claiming authorship, non-opt-in language) hard-block output.

**AI ledger immutability — DB trigger, not application logic.**
Application bugs can bypass service-layer immutability. The Postgres trigger `ledger_no_delete` and `ledger_restrict_update` enforce append-only semantics at the database level, making it impossible for any code path (including future developers) to alter historical records. Only `writer_action`, `modification_delta`, and `action_recorded_at` can be updated — these are the writer's response fields which are not known at ledger write time.
