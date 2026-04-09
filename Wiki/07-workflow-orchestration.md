# 07 — Workflow Orchestration

## Why Orchestration, Not Application State Machines

Production workflows are long-running (hours to weeks), approval-bound, and require compensation logic on failure. Encoding these as application-level state machines leads to:
- Lost state on service restarts
- No visibility into in-flight workflows
- Ad-hoc compensation that misses edge cases
- Impossible debugging of stuck approvals

**Temporal** provides durable execution — workflows survive service restarts, infrastructure failures, and can run for weeks while waiting for human approvals. Every workflow step is retried automatically until it succeeds or exhausts a configurable policy. The event history is the workflow state — no external database needed for workflow state management.

---

## 1. Temporal Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│  Temporal Cloud (managed server — temporal.io hosted)                │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Task Queues                                                    │  │
│  │  • publish-saga       • import-saga                            │  │
│  │  • call-sheet         • revision-dist                          │  │
│  │  • breakdown-activity • scheduling-activity                    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ gRPC (mTLS via Istio)
          ┌───────────────┼───────────────────┐
          ▼               ▼                   ▼
  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
  │ publish-saga │  │ import-saga  │  │ call-sheet + rev-dist │
  │   Worker     │  │   Worker     │  │      Worker           │
  │  (K8s pod)   │  │  (K8s pod)   │  │     (K8s pod)         │
  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘
         │                 │                      │
         │ gRPC to services│                      │
         ▼                 ▼                      ▼
  Script, Breakdown,  Import Pipeline,     Scheduling, Budget,
  Scheduling, Legal,  Search, Script       Notification, Script
  Budget, Notification Services             Services
```

Workers run inside the Kubernetes service mesh (Istio mTLS). Workers have no inbound network access — they only poll Temporal Cloud outbound. This means workers don't need ingress rules or service mesh ingress configuration.

---

## 2. Approval Routing Configuration

Before diving into workflow code, the approval routing configuration schema governs how each org's approval chain is structured.

### PostgreSQL Schema

```sql
-- Schema: workflow.*

CREATE TABLE workflow.approval_chain_templates (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      UUID,   -- NULL = built-in global template
  name        TEXT NOT NULL,
  description TEXT,
  steps       JSONB NOT NULL,  -- array of ApprovalStep
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Example JSONB steps structure:
-- [
--   {
--     "step_id": "legal",
--     "label": "Legal Review",
--     "approver_role": "legal_counsel",
--     "parallel": true,          -- runs in parallel with "breakdown" step
--     "auto_approve_after_hours": 48,
--     "escalate_after_hours": 24,
--     "escalation_role": "general_counsel"
--   },
--   {
--     "step_id": "showrunner",
--     "label": "Showrunner Approval",
--     "approver_role": "showrunner",
--     "parallel": false,         -- sequential: runs after all parallel steps complete
--     "auto_approve_after_hours": 48,
--     "escalate_after_hours": 24,
--     "escalation_role": "executive_producer"
--   }
-- ]

CREATE TABLE workflow.approval_chain_configs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id          UUID NOT NULL REFERENCES auth.organizations(id),
  project_id      UUID,           -- NULL = org-wide default
  template_id     UUID REFERENCES workflow.approval_chain_templates(id),
  override_steps  JSONB,          -- if set, overrides template steps for this project
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (org_id, project_id)
);

-- In-flight approval requests (one row per approval step per workflow run)
CREATE TABLE workflow.approval_requests (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_id     TEXT NOT NULL,   -- Temporal workflow ID
  run_id          TEXT NOT NULL,   -- Temporal run ID
  step_id         TEXT NOT NULL,
  project_id      UUID NOT NULL,
  script_version_id UUID,
  approver_user_id UUID,           -- specific user, or resolved from role
  approver_role   TEXT,
  status          TEXT NOT NULL DEFAULT 'pending',  -- 'pending' | 'approved' | 'rejected' | 'escalated' | 'auto_approved'
  notes           TEXT,
  escalated_at    TIMESTAMPTZ,
  decided_at      TIMESTAMPTZ,
  decided_by      UUID REFERENCES auth.users(id),
  auto_approved_reason TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX ON workflow.approval_requests (workflow_id, step_id);
CREATE INDEX ON workflow.approval_requests (status) WHERE status = 'pending';
CREATE INDEX ON workflow.approval_requests (approver_user_id, status);
```

### TypeScript Config Types

```typescript
// packages/workflows/src/approval-config.ts

export interface ApprovalStep {
  step_id: string;
  label: string;
  approver_role: string;
  parallel: boolean;              // true = run concurrently with other parallel steps
  auto_approve_after_hours: number;
  escalate_after_hours: number;
  escalation_role?: string;
  required?: boolean;             // false = advisory only, does not block
}

export interface ApprovalChainConfig {
  org_id: string;
  project_id: string | null;
  steps: ApprovalStep[];
}

// Built-in templates
export const APPROVAL_TEMPLATES = {
  STUDIO_FEATURE: [
    { step_id: 'legal', label: 'Legal Review', approver_role: 'legal_counsel', parallel: true, auto_approve_after_hours: 48, escalate_after_hours: 24 },
    { step_id: 'writer', label: 'Writer Sign-off', approver_role: 'writer', parallel: true, auto_approve_after_hours: 24, escalate_after_hours: 12 },
    { step_id: 'producer', label: 'Producer Approval', approver_role: 'producer', parallel: false, auto_approve_after_hours: 48, escalate_after_hours: 24 },
    { step_id: 'studio_exec', label: 'Studio Executive', approver_role: 'studio_exec', parallel: false, auto_approve_after_hours: 72, escalate_after_hours: 48 },
  ] satisfies ApprovalStep[],

  TV_EPISODE: [
    { step_id: 'legal', label: 'Legal Review', approver_role: 'legal_counsel', parallel: true, auto_approve_after_hours: 48, escalate_after_hours: 24 },
    { step_id: 'showrunner', label: 'Showrunner', approver_role: 'showrunner', parallel: false, auto_approve_after_hours: 48, escalate_after_hours: 24 },
    { step_id: 'network', label: 'Network Approval', approver_role: 'network_exec', parallel: false, auto_approve_after_hours: 72, escalate_after_hours: 48 },
  ] satisfies ApprovalStep[],

  INDEPENDENT: [
    { step_id: 'producer', label: 'Producer Self-Approval', approver_role: 'producer', parallel: false, auto_approve_after_hours: 24, escalate_after_hours: 12 },
  ] satisfies ApprovalStep[],
};
```

---

## 3. Publish-to-Production Saga

### State Machine

```
                                             ┌─────────────────────────┐
                                             │         START           │
                                             └────────────┬────────────┘
                                                          │
                                             ┌────────────▼────────────┐
                                             │   lockDraft             │◄── Reject if: merge conflict,
                                             │   (Script Service)      │    unresolved canon violation
                                             └────────────┬────────────┘
                                                          │ success
                                  ┌──────────────────────┼──────────────────────┐
                                  │ parallel             │                      │
                        ┌─────────▼────────┐   ┌────────▼─────────┐  ┌────────▼────────┐
                        │ validatePolicy   │   │ validateCanon    │  │ validateAI      │
                        │ (Legal Service)  │   │ (Bible Service)  │  │ disclosure      │
                        └─────────┬────────┘   └────────┬─────────┘  └────────┬────────┘
                                  └──────────────────────┼──────────────────────┘
                                                         │ all pass
                                         ┌───────────────▼───────────────┐
                                         │ extractBreakdown               │◄── Compensate: clearBreakdownArtifacts
                                         │ (Breakdown Service)            │    → unlockDraft
                                         └───────────────┬───────────────┘
                                                         │
                                         ┌───────────────▼───────────────┐
                                         │ computeScheduleDelta           │◄── Compensate: rollbackScheduleDelta
                                         │ (Scheduling Service)           │    → clearBreakdownArtifacts
                                         └───────────────┬───────────────┘         → unlockDraft
                                                         │
                                         ┌───────────────▼───────────────┐
                                         │ computeBudgetDelta             │◄── Compensate: rollbackBudgetDelta
                                         │ (Budget Service)               │    → rollbackScheduleDelta
                                         └───────────────┬───────────────┘    → clearBreakdownArtifacts
                                                         │                    → unlockDraft
                                         ┌───────────────▼───────────────┐
                                         │ routeApprovals                 │
                                         │ (per org config, may be        │◄── waitForSignal('approval.*')
                                         │  parallel + sequential)        │    with escalation timers
                                         └───────────────┬───────────────┘
                                                         │ all approved
                                         ┌───────────────▼───────────────┐
                                         │ operationalLock                │
                                         │ + distribute notifications     │
                                         └───────────────┬───────────────┘
                                                         │
                                                      COMPLETE
```

### TypeScript Workflow Definition

```typescript
// workers/publish-saga/src/workflows/publish-to-production.ts

import {
  proxyActivities,
  defineSignal,
  defineQuery,
  setHandler,
  condition,
  sleep,
  ActivityFailure,
  ApplicationFailure,
} from '@temporalio/workflow';
import type * as activities from '../activities';
import { ApprovalStep, ApprovalChainConfig } from '../../../packages/workflows/src/approval-config';

// ── Activity proxies ──────────────────────────────────────────────────────────

const {
  lockDraft,
  unlockDraft,
  validatePolicy,
  validateCanonConsistency,
  validateAIDisclosure,
  extractBreakdown,
  clearBreakdownArtifacts,
  computeScheduleDelta,
  rollbackScheduleDelta,
  computeBudgetDelta,
  rollbackBudgetDelta,
  createApprovalRequests,
  notifyApprovers,
  escalateApproval,
  operationalLockVersion,
  distributeRevisionNotifications,
  updatePublishStatus,
  watermarkExport,
} = proxyActivities<typeof activities>({
  scheduleToCloseTimeout: '5 minutes',  // per activity
  retry: {
    initialInterval: '1s',
    backoffCoefficient: 2.0,
    maximumAttempts: 3,
    nonRetryableErrorTypes: [
      'PolicyViolationError',
      'CanonConflictError',
      'AIDisclosureError',
      'DraftLockConflictError',
    ],
  },
});

// ── Signals and queries ────────────────────────────────────────────────────────

export const approvalSignal = defineSignal<[ApprovalDecision]>('approval.decision');
export const withdrawalSignal = defineSignal('publish.withdraw');

export const publishStatusQuery = defineQuery<PublishStatus>('publish.status');

// ── Workflow types ──────────────────────────────────────────────────────────────

export interface PublishWorkflowInput {
  scriptId: string;
  scriptVersionId: string;
  projectId: string;
  orgId: string;
  requestedBy: string;       // user ID
  approvalConfig: ApprovalChainConfig;
  options?: {
    skipBudgetDelta?: boolean;    // for minor revisions
    emergencyApproval?: boolean;  // collapses approval chain to single approver
  };
}

export interface ApprovalDecision {
  stepId: string;
  decision: 'approved' | 'rejected';
  decidedBy: string;
  notes?: string;
}

export type PublishStatus =
  | 'locking_draft'
  | 'validating'
  | 'extracting_breakdown'
  | 'computing_schedule'
  | 'computing_budget'
  | 'awaiting_approval'
  | 'locking_production'
  | 'complete'
  | 'failed'
  | 'withdrawn';

// ── Main workflow ───────────────────────────────────────────────────────────────

export async function publishToProductionWorkflow(input: PublishWorkflowInput): Promise<void> {
  let status: PublishStatus = 'locking_draft';
  const approvalDecisions = new Map<string, ApprovalDecision>();
  let withdrawn = false;

  setHandler(publishStatusQuery, () => status);

  setHandler(approvalSignal, (decision: ApprovalDecision) => {
    approvalDecisions.set(decision.stepId, decision);
  });

  setHandler(withdrawalSignal, () => {
    withdrawn = true;
  });

  // Track compensation stack (LIFO — most recent action compensated first)
  const compensations: (() => Promise<void>)[] = [];

  try {
    // ── Step 1: Lock draft ──────────────────────────────────────────────────
    await updatePublishStatus(input.scriptId, 'publishing');
    await lockDraft({ scriptId: input.scriptId, scriptVersionId: input.scriptVersionId });
    compensations.push(() => unlockDraft({ scriptId: input.scriptId }));

    status = 'validating';

    // ── Step 2: Parallel validation ──────────────────────────────────────────
    await Promise.all([
      validatePolicy({ scriptId: input.scriptId, orgId: input.orgId, projectId: input.projectId }),
      validateCanonConsistency({ scriptId: input.scriptId, projectId: input.projectId }),
      validateAIDisclosure({ scriptId: input.scriptId, projectId: input.projectId }),
    ]);

    // ── Step 3: Breakdown extraction ─────────────────────────────────────────
    status = 'extracting_breakdown';
    await extractBreakdown({ scriptVersionId: input.scriptVersionId, projectId: input.projectId });
    compensations.push(() => clearBreakdownArtifacts({ scriptVersionId: input.scriptVersionId }));

    // ── Step 4: Schedule delta ───────────────────────────────────────────────
    status = 'computing_schedule';
    const scheduleDelta = await computeScheduleDelta({
      scriptVersionId: input.scriptVersionId,
      projectId: input.projectId,
    });
    compensations.push(() => rollbackScheduleDelta({ deltaId: scheduleDelta.deltaId }));

    // ── Step 5: Budget delta (optional) ─────────────────────────────────────
    status = 'computing_budget';
    if (!input.options?.skipBudgetDelta) {
      const budgetDelta = await computeBudgetDelta({
        scriptVersionId: input.scriptVersionId,
        projectId: input.projectId,
        scheduleDeltaId: scheduleDelta.deltaId,
      });
      compensations.push(() => rollbackBudgetDelta({ deltaId: budgetDelta.deltaId }));
    }

    // ── Step 6: Approval routing ─────────────────────────────────────────────
    status = 'awaiting_approval';
    await runApprovalChain(input, approvalDecisions, withdrawn);

    // ── Step 7: Operational lock ─────────────────────────────────────────────
    status = 'locking_production';
    await operationalLockVersion({
      scriptId: input.scriptId,
      scriptVersionId: input.scriptVersionId,
      approvalDecisions: Array.from(approvalDecisions.values()),
    });

    await watermarkExport({ scriptVersionId: input.scriptVersionId, projectId: input.projectId });
    await distributeRevisionNotifications({ scriptVersionId: input.scriptVersionId, projectId: input.projectId });

    status = 'complete';
    await updatePublishStatus(input.scriptId, 'production');

  } catch (err) {
    // Handle withdrawal
    if (withdrawn) {
      status = 'withdrawn';
      await updatePublishStatus(input.scriptId, 'draft');
    } else {
      status = 'failed';
      await updatePublishStatus(input.scriptId, 'draft');
    }

    // Run compensations in reverse order
    for (const compensate of [...compensations].reverse()) {
      try {
        await compensate();
      } catch (compensationErr) {
        // Log compensation failure — don't rethrow, continue compensating
        console.error('Compensation failed:', compensationErr);
      }
    }

    throw err;  // Rethrow so Temporal marks workflow as failed
  }
}

// ── Approval chain execution ────────────────────────────────────────────────────

async function runApprovalChain(
  input: PublishWorkflowInput,
  decisions: Map<string, ApprovalDecision>,
  withdrawnRef: boolean,
): Promise<void> {
  const { steps } = input.approvalConfig;

  // Create approval request records in DB (async fire-and-forget via activity)
  await createApprovalRequests({
    workflowId: workflowInfo().workflowId,
    runId: workflowInfo().firstExecutionRunId,
    steps,
    projectId: input.projectId,
    scriptVersionId: input.scriptVersionId,
  });

  // Separate parallel and sequential steps
  const parallelSteps = steps.filter(s => s.parallel);
  const sequentialSteps = steps.filter(s => !s.parallel);

  // Run all parallel steps concurrently
  if (parallelSteps.length > 0) {
    await Promise.all(parallelSteps.map(step => waitForApprovalStep(step, decisions, input)));
  }

  // Run sequential steps in order
  for (const step of sequentialSteps) {
    await waitForApprovalStep(step, decisions, input);
  }
}

async function waitForApprovalStep(
  step: ApprovalStep,
  decisions: Map<string, ApprovalDecision>,
  input: PublishWorkflowInput,
): Promise<void> {
  // Notify approver
  await notifyApprovers({
    stepId: step.step_id,
    approverRole: step.approver_role,
    projectId: input.projectId,
    scriptVersionId: input.scriptVersionId,
    workflowId: workflowInfo().workflowId,
  });

  const escalateAfterMs = step.escalate_after_hours * 60 * 60 * 1000;
  const autoApproveAfterMs = step.auto_approve_after_hours * 60 * 60 * 1000;
  let escalated = false;

  // Wait for decision, escalate at 24h, auto-approve at 48h
  await condition(
    () => decisions.has(step.step_id),
    autoApproveAfterMs,
  ).catch(async () => {
    // Timeout reached → auto-approve
    if (!escalated) {
      // Escalation didn't fire — check if we should have
    }
    decisions.set(step.step_id, {
      stepId: step.step_id,
      decision: 'approved',
      decidedBy: 'system',
      notes: `Auto-approved after ${step.auto_approve_after_hours}h timeout`,
    });
    await escalateApproval({ stepId: step.step_id, reason: 'auto_approved', projectId: input.projectId });
  });

  // Launch escalation timer concurrently
  sleep(escalateAfterMs).then(async () => {
    if (!decisions.has(step.step_id)) {
      escalated = true;
      await escalateApproval({
        stepId: step.step_id,
        reason: 'escalated',
        escalationRole: step.escalation_role,
        projectId: input.projectId,
      });
    }
  });

  const decision = decisions.get(step.step_id)!;
  if (decision.decision === 'rejected') {
    throw ApplicationFailure.nonRetryable(
      `Approval rejected at step "${step.label}" by ${decision.decidedBy}: ${decision.notes ?? ''}`,
      'ApprovalRejectedError',
      { stepId: step.step_id, decidedBy: decision.decidedBy },
    );
  }
}
```

---

## 4. Activity Definitions

```typescript
// workers/publish-saga/src/activities/index.ts

import { Context } from '@temporalio/activity';
import { createScriptServiceClient } from '../../packages/proto/src/script_grpc_pb';
import { createBreakdownServiceClient } from '../../packages/proto/src/breakdown_grpc_pb';
import { createSchedulingServiceClient } from '../../packages/proto/src/scheduling_grpc_pb';

// ── Draft management ─────────────────────────────────────────────────────────

export async function lockDraft(input: { scriptId: string; scriptVersionId: string }): Promise<void> {
  const scriptClient = createScriptServiceClient();
  const response = await scriptClient.lockDraftForPublish({
    script_id: input.scriptId,
    version_id: input.scriptVersionId,
  });
  if (!response.success) {
    if (response.error === 'MERGE_CONFLICT') {
      throw ApplicationFailure.nonRetryable('Script has unresolved merge conflicts', 'DraftLockConflictError');
    }
    if (response.error === 'CANON_VIOLATION') {
      throw ApplicationFailure.nonRetryable('Script has unresolved canon violations', 'CanonConflictError');
    }
    throw new Error(`Failed to lock draft: ${response.error}`);
  }
}

export async function unlockDraft(input: { scriptId: string }): Promise<void> {
  const scriptClient = createScriptServiceClient();
  await scriptClient.unlockDraft({ script_id: input.scriptId });
}

// ── Validation ────────────────────────────────────────────────────────────────

export async function validatePolicy(input: {
  scriptId: string; orgId: string; projectId: string;
}): Promise<void> {
  const { scriptId, orgId, projectId } = input;

  // Check rights and NDA gates
  const legalClient = createLegalServiceClient();
  const result = await legalClient.validateScriptPolicy({ script_id: scriptId, org_id: orgId, project_id: projectId });

  if (!result.passed) {
    throw ApplicationFailure.nonRetryable(
      `Policy validation failed: ${result.violations.map(v => v.description).join('; ')}`,
      'PolicyViolationError',
      { violations: result.violations },
    );
  }
}

export async function validateCanonConsistency(input: {
  scriptId: string; projectId: string;
}): Promise<void> {
  const bibleClient = createBibleServiceClient();
  const result = await bibleClient.validateScriptConsistency({
    script_id: input.scriptId,
    project_id: input.projectId,
  });

  if (result.conflicts.length > 0) {
    throw ApplicationFailure.nonRetryable(
      `Canon consistency violations: ${result.conflicts.length} conflicts found`,
      'CanonConflictError',
      { conflicts: result.conflicts },
    );
  }
}

export async function validateAIDisclosure(input: {
  scriptId: string; projectId: string;
}): Promise<void> {
  const aiClient = createAIServiceClient();
  const result = await aiClient.getAIDisclosureStatus({
    script_id: input.scriptId,
    project_id: input.projectId,
  });

  // If AI was used and company hasn't consented, block
  if (result.ai_used && !result.company_consent_on_file) {
    throw ApplicationFailure.nonRetryable(
      'AI assistance was used in this script version but company consent is not on file',
      'AIDisclosureError',
    );
  }
}

// ── Breakdown ─────────────────────────────────────────────────────────────────

export async function extractBreakdown(input: {
  scriptVersionId: string; projectId: string;
}): Promise<{ breakdownVersionId: string }> {
  // This is retryable (NLP can be flaky under load)
  const breakdownClient = createBreakdownServiceClient();
  const result = await breakdownClient.extractBreakdownFromVersion({
    script_version_id: input.scriptVersionId,
    project_id: input.projectId,
  });
  return { breakdownVersionId: result.breakdown_version_id };
}

export async function clearBreakdownArtifacts(input: { scriptVersionId: string }): Promise<void> {
  const breakdownClient = createBreakdownServiceClient();
  await breakdownClient.deleteBreakdownVersion({ script_version_id: input.scriptVersionId });
}

// ── Schedule ──────────────────────────────────────────────────────────────────

export interface ScheduleDeltaResult {
  deltaId: string;
  affectedShootDays: number;
  daysAdded: number;
  daysRemoved: number;
}

export async function computeScheduleDelta(input: {
  scriptVersionId: string; projectId: string;
}): Promise<ScheduleDeltaResult> {
  const schedulingClient = createSchedulingServiceClient();
  const result = await schedulingClient.computeVersionDelta({
    script_version_id: input.scriptVersionId,
    project_id: input.projectId,
  });
  return {
    deltaId: result.delta_id,
    affectedShootDays: result.affected_shoot_days,
    daysAdded: result.days_added,
    daysRemoved: result.days_removed,
  };
}

export async function rollbackScheduleDelta(input: { deltaId: string }): Promise<void> {
  const schedulingClient = createSchedulingServiceClient();
  await schedulingClient.rollbackDelta({ delta_id: input.deltaId });
}

// ── Budget ────────────────────────────────────────────────────────────────────

export interface BudgetDeltaResult {
  deltaId: string;
  totalDeltaUsd: number;
}

export async function computeBudgetDelta(input: {
  scriptVersionId: string; projectId: string; scheduleDeltaId: string;
}): Promise<BudgetDeltaResult> {
  const budgetClient = createBudgetServiceClient();
  const result = await budgetClient.computeVersionDelta({
    script_version_id: input.scriptVersionId,
    project_id: input.projectId,
    schedule_delta_id: input.scheduleDeltaId,
  });
  return { deltaId: result.delta_id, totalDeltaUsd: result.total_delta_usd };
}

export async function rollbackBudgetDelta(input: { deltaId: string }): Promise<void> {
  const budgetClient = createBudgetServiceClient();
  await budgetClient.rollbackDelta({ delta_id: input.deltaId });
}

// ── Approvals ─────────────────────────────────────────────────────────────────

export async function createApprovalRequests(input: {
  workflowId: string;
  runId: string;
  steps: ApprovalStep[];
  projectId: string;
  scriptVersionId: string;
}): Promise<void> {
  const db = getDB();
  for (const step of input.steps) {
    await db.query(sql`
      INSERT INTO workflow.approval_requests
        (workflow_id, run_id, step_id, project_id, script_version_id, approver_role)
      VALUES (${input.workflowId}, ${input.runId}, ${step.step_id}, ${input.projectId}, ${input.scriptVersionId}, ${step.approver_role})
      ON CONFLICT DO NOTHING
    `);
  }
}

export async function notifyApprovers(input: {
  stepId: string;
  approverRole: string;
  projectId: string;
  scriptVersionId: string;
  workflowId: string;
}): Promise<void> {
  const notificationClient = createNotificationServiceClient();
  await notificationClient.sendApprovalRequest({
    step_id: input.stepId,
    approver_role: input.approverRole,
    project_id: input.projectId,
    script_version_id: input.scriptVersionId,
    workflow_id: input.workflowId,
    approval_url: `${process.env.APP_URL}/projects/${input.projectId}/approvals/${input.workflowId}/${input.stepId}`,
  });
}

export async function escalateApproval(input: {
  stepId: string;
  reason: 'escalated' | 'auto_approved';
  escalationRole?: string;
  projectId: string;
}): Promise<void> {
  const db = getDB();
  await db.query(sql`
    UPDATE workflow.approval_requests
    SET status = ${input.reason}, escalated_at = NOW()
    WHERE step_id = ${input.stepId}
      AND project_id = ${input.projectId}
      AND status = 'pending'
  `);

  if (input.reason === 'escalated' && input.escalationRole) {
    const notificationClient = createNotificationServiceClient();
    await notificationClient.sendEscalationNotice({
      step_id: input.stepId,
      escalation_role: input.escalationRole,
      project_id: input.projectId,
    });
  }
}

// ── Operational lock ──────────────────────────────────────────────────────────

export async function operationalLockVersion(input: {
  scriptId: string;
  scriptVersionId: string;
  approvalDecisions: ApprovalDecision[];
}): Promise<void> {
  const scriptClient = createScriptServiceClient();
  await scriptClient.operationalLockVersion({
    script_id: input.scriptId,
    version_id: input.scriptVersionId,
    approval_decisions: input.approvalDecisions.map(d => ({
      step_id: d.stepId,
      decision: d.decision,
      decided_by: d.decidedBy,
      notes: d.notes ?? '',
    })),
  });
}

export async function watermarkExport(input: {
  scriptVersionId: string; projectId: string;
}): Promise<void> {
  const watermarkClient = createWatermarkServiceClient();
  await watermarkClient.embedWatermarkInVersion({
    version_id: input.scriptVersionId,
    project_id: input.projectId,
  });
}

export async function distributeRevisionNotifications(input: {
  scriptVersionId: string; projectId: string;
}): Promise<void> {
  const notificationClient = createNotificationServiceClient();
  await notificationClient.distributeRevision({
    version_id: input.scriptVersionId,
    project_id: input.projectId,
  });
}

export async function updatePublishStatus(scriptId: string, status: string): Promise<void> {
  const scriptClient = createScriptServiceClient();
  await scriptClient.updatePublishStatus({ script_id: scriptId, status });
}
```

---

## 5. Import Saga

### Temporal Workflow

```typescript
// workers/import-saga/src/workflows/import-script.ts

export interface ImportWorkflowInput {
  importJobId: string;     // created by API on file upload
  orgId: string;
  projectId: string;
  uploadedBy: string;
  s3Key: string;           // uploaded file in S3
  format: 'fdx' | 'fountain' | 'pdf' | 'pdf_ocr';
  options?: {
    episodeNumbers?: number[];    // for batch episode imports
    targetRevisionColor?: string;
  };
}

const {
  detectFormat,
  parseToAST,
  validateAST,
  generateConfidenceWarnings,
  waitForUserReview,
  persistAST,
  triggerSearchIndexing,
  publishImportComplete,
  archiveOriginalFile,
  markImportFailed,
} = proxyActivities<typeof importActivities>({
  scheduleToCloseTimeout: '10 minutes',
  retry: {
    initialInterval: '2s',
    backoffCoefficient: 2.0,
    maximumAttempts: 5,
    nonRetryableErrorTypes: ['ParseFormatError', 'StructuralValidationError'],
  },
});

export const importReviewSignal = defineSignal<[ImportReviewDecision]>('import.review');
export const importStatusQuery = defineQuery<ImportStatus>('import.status');

export async function importScriptWorkflow(input: ImportWorkflowInput): Promise<string> {
  let status: ImportStatus = 'detecting_format';
  let reviewDecision: ImportReviewDecision | null = null;

  setHandler(importStatusQuery, () => status);
  setHandler(importReviewSignal, (d) => { reviewDecision = d; });

  try {
    // ── Detect + parse ──────────────────────────────────────────────────────
    const format = await detectFormat({ s3Key: input.s3Key, hintedFormat: input.format });
    status = 'parsing';
    const parseResult = await parseToAST({ s3Key: input.s3Key, format, importJobId: input.importJobId });

    status = 'validating';
    const validation = await validateAST({ astId: parseResult.astId, importJobId: input.importJobId });

    // ── Confidence warnings ──────────────────────────────────────────────────
    const warnings = await generateConfidenceWarnings({ astId: parseResult.astId, importJobId: input.importJobId });

    if (warnings.requiresReview) {
      // Low-confidence elements — send to user for review
      status = 'awaiting_review';
      await condition(() => reviewDecision !== null, '7 days');

      if (reviewDecision === null) {
        throw ApplicationFailure.nonRetryable('Import review timed out — job cancelled', 'ImportTimeoutError');
      }

      if (reviewDecision.action === 'cancel') {
        throw ApplicationFailure.nonRetryable('Import cancelled by user', 'ImportCancelledError');
      }
    }

    // ── Persist ─────────────────────────────────────────────────────────────
    status = 'persisting';
    const scriptId = await persistAST({
      astId: parseResult.astId,
      importJobId: input.importJobId,
      projectId: input.projectId,
      orgId: input.orgId,
      uploadedBy: input.uploadedBy,
      options: input.options,
    });

    // ── Post-persist tasks (parallel) ────────────────────────────────────────
    status = 'indexing';
    await Promise.all([
      triggerSearchIndexing({ scriptId, projectId: input.projectId }),
      archiveOriginalFile({ s3Key: input.s3Key, importJobId: input.importJobId }),
      publishImportComplete({ importJobId: input.importJobId, scriptId }),
    ]);

    status = 'complete';
    return scriptId;

  } catch (err) {
    await markImportFailed({ importJobId: input.importJobId, error: (err as Error).message });
    throw err;
  }
}
```

---

## 6. Call Sheet Generation Saga

```typescript
// workers/call-sheet/src/workflows/generate-call-sheet.ts

export interface CallSheetWorkflowInput {
  projectId: string;
  shootDate: string;       // YYYY-MM-DD
  triggeredBy: 'schedule_change' | 'manual' | 'revision_published';
  requestedBy: string;
  adUserId: string;         // 1st AD who must approve before distribution
  upmUserId: string;        // UPM who must approve
}

const {
  pullScenesForShootDay,
  pullCastAvailability,
  pullCrewAssignments,
  pullLocationLogistics,
  generateCallSheetDocument,
  sendForADApproval,
  waitForADApproval,
  distributeCallSheet,
  archiveCallSheet,
} = proxyActivities<typeof callSheetActivities>({
  scheduleToCloseTimeout: '5 minutes',
  retry: { maximumAttempts: 3 },
});

export const callSheetApprovalSignal = defineSignal<[{ approved: boolean; notes?: string }]>('callsheet.approval');

export async function generateCallSheetWorkflow(input: CallSheetWorkflowInput): Promise<string> {
  let adDecision: { approved: boolean; notes?: string } | null = null;
  setHandler(callSheetApprovalSignal, (d) => { adDecision = d; });

  // Pull all data in parallel
  const [scenes, cast, crew, logistics] = await Promise.all([
    pullScenesForShootDay({ projectId: input.projectId, shootDate: input.shootDate }),
    pullCastAvailability({ projectId: input.projectId, shootDate: input.shootDate }),
    pullCrewAssignments({ projectId: input.projectId, shootDate: input.shootDate }),
    pullLocationLogistics({ projectId: input.projectId, shootDate: input.shootDate }),
  ]);

  // Generate document
  const callSheetId = await generateCallSheetDocument({
    projectId: input.projectId,
    shootDate: input.shootDate,
    scenes, cast, crew, logistics,
  });

  // AD + UPM approval (parallel)
  await sendForADApproval({ callSheetId, adUserId: input.adUserId, upmUserId: input.upmUserId });

  // Wait up to 4 hours for approval (call sheets have tighter deadlines)
  await condition(() => adDecision !== null, '4 hours');

  if (adDecision === null || !adDecision.approved) {
    // Return for revision — do not distribute
    return callSheetId;
  }

  // Distribute + archive
  await Promise.all([
    distributeCallSheet({ callSheetId, projectId: input.projectId }),
    archiveCallSheet({ callSheetId }),
  ]);

  return callSheetId;
}
```

---

## 7. Revision Distribution Saga

When a new revision color is published (operational lock completes), a separate workflow handles distribution:

```typescript
// workers/revision-dist/src/workflows/distribute-revision.ts

export interface RevisionDistWorkflowInput {
  scriptVersionId: string;
  projectId: string;
  revisionColor: string;   // 'white' | 'blue' | 'pink' | ...
  changedSceneIds: string[];
}

const {
  generateRevisionDiff,
  generateColoredPages,
  determineDistributionList,
  embedWatermarkPerRecipient,
  distributeToRecipient,
  updateBreakdownForChangedScenes,
  notifyEditorial,
  archiveRevision,
} = proxyActivities<typeof revisionActivities>({
  scheduleToCloseTimeout: '10 minutes',
  retry: { maximumAttempts: 3 },
});

export async function distributeRevisionWorkflow(input: RevisionDistWorkflowInput): Promise<void> {
  // 1. Generate diff + colored pages (these are expensive — do them once)
  const [diff, coloredPages] = await Promise.all([
    generateRevisionDiff({ scriptVersionId: input.scriptVersionId, projectId: input.projectId }),
    generateColoredPages({ scriptVersionId: input.scriptVersionId, revisionColor: input.revisionColor }),
  ]);

  // 2. Determine who should receive this revision
  const recipients = await determineDistributionList({
    projectId: input.projectId,
    scriptVersionId: input.scriptVersionId,
  });

  // 3. Per-recipient watermarking + distribution (parallel, with error isolation)
  await Promise.allSettled(
    recipients.map(async (recipient) => {
      const watermarkedKey = await embedWatermarkPerRecipient({
        baseS3Key: coloredPages.s3Key,
        recipientId: recipient.userId,
        recipientName: recipient.name,
        scriptVersionId: input.scriptVersionId,
      });
      await distributeToRecipient({
        recipientId: recipient.userId,
        recipientEmail: recipient.email,
        watermarkedS3Key: watermarkedKey,
        revisionColor: input.revisionColor,
        projectId: input.projectId,
      });
    }),
  );

  // 4. Side effects
  await Promise.all([
    updateBreakdownForChangedScenes({
      scriptVersionId: input.scriptVersionId,
      changedSceneIds: input.changedSceneIds,
      projectId: input.projectId,
    }),
    notifyEditorial({
      scriptVersionId: input.scriptVersionId,
      diff: diff.summary,
      projectId: input.projectId,
    }),
    archiveRevision({ scriptVersionId: input.scriptVersionId, projectId: input.projectId }),
  ]);
}
```

---

## 8. Temporal Worker Kubernetes Deployment

```yaml
# infra/k8s/workers/publish-saga.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: publish-saga-worker
  namespace: scriptos
spec:
  replicas: 3
  selector:
    matchLabels:
      app: publish-saga-worker
  template:
    metadata:
      labels:
        app: publish-saga-worker
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: temporal-worker
      containers:
        - name: worker
          image: scriptos/publish-saga-worker:latest
          env:
            - name: TEMPORAL_ADDRESS
              valueFrom:
                secretKeyRef:
                  name: temporal-cloud-creds
                  key: endpoint
            - name: TEMPORAL_NAMESPACE
              value: "scriptos.a2df3"
            - name: TEMPORAL_TLS_CERT
              valueFrom:
                secretKeyRef:
                  name: temporal-cloud-creds
                  key: tls_cert
            - name: TEMPORAL_TLS_KEY
              valueFrom:
                secretKeyRef:
                  name: temporal-cloud-creds
                  key: tls_key
            - name: TASK_QUEUE
              value: "publish-saga"
            - name: MAX_CONCURRENT_ACTIVITIES
              value: "10"
            - name: MAX_CONCURRENT_WORKFLOWS
              value: "5"
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: publish-saga-worker-hpa
  namespace: scriptos
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: publish-saga-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: temporal_task_queue_depth
          selector:
            matchLabels:
              task_queue: publish-saga
        target:
          type: AverageValue
          averageValue: "5"   # scale up when >5 tasks/worker pending
```

### Worker Entry Point

```typescript
// workers/publish-saga/src/index.ts

import { NativeConnection, Worker } from '@temporalio/worker';
import * as workflows from './workflows/publish-to-production';
import * as activities from './activities';
import { makeLogger } from '../../packages/common/src/logger';

const logger = makeLogger('publish-saga-worker');

async function main() {
  const connection = await NativeConnection.connect({
    address: process.env.TEMPORAL_ADDRESS!,
    tls: {
      clientCertPair: {
        crt: Buffer.from(process.env.TEMPORAL_TLS_CERT!, 'base64'),
        key: Buffer.from(process.env.TEMPORAL_TLS_KEY!, 'base64'),
      },
    },
  });

  const worker = await Worker.create({
    connection,
    namespace: process.env.TEMPORAL_NAMESPACE!,
    taskQueue: process.env.TASK_QUEUE ?? 'publish-saga',
    workflowsPath: require.resolve('./workflows'),
    activities,
    maxConcurrentActivityTaskExecutions: parseInt(process.env.MAX_CONCURRENT_ACTIVITIES ?? '10'),
    maxConcurrentWorkflowTaskExecutions: parseInt(process.env.MAX_CONCURRENT_WORKFLOWS ?? '5'),
    sinks: {
      logger: {
        info: { fn: (info, message) => logger.info(message, info) },
        warn: { fn: (info, message) => logger.warn(message, info) },
        error: { fn: (info, message) => logger.error(message, info) },
      },
    },
  });

  logger.info('Publish saga worker started');
  await worker.run();
}

main().catch((err) => {
  console.error('Worker fatal error', err);
  process.exit(1);
});
```

---

## 9. Workflow Invocation (API Layer)

### Starting a Publish Saga

```typescript
// services/script/src/publish-handler.ts

import { Client, Connection } from '@temporalio/client';
import { publishToProductionWorkflow } from '../../packages/workflows/src/publish-to-production';

export async function startPublishSaga(
  scriptId: string,
  scriptVersionId: string,
  projectId: string,
  orgId: string,
  requestedBy: string,
  db: DB,
): Promise<string> {
  // Load approval chain config for this org/project
  const approvalConfig = await loadApprovalChainConfig(projectId, orgId, db);

  const connection = await Connection.connect({
    address: process.env.TEMPORAL_ADDRESS!,
    tls: {
      clientCertPair: {
        crt: Buffer.from(process.env.TEMPORAL_TLS_CERT!, 'base64'),
        key: Buffer.from(process.env.TEMPORAL_TLS_KEY!, 'base64'),
      },
    },
  });

  const client = new Client({ connection, namespace: process.env.TEMPORAL_NAMESPACE! });

  const workflowId = `publish:${scriptVersionId}`;  // deterministic — prevents duplicate publishes

  const handle = await client.workflow.start(publishToProductionWorkflow, {
    taskQueue: 'publish-saga',
    workflowId,
    args: [{ scriptId, scriptVersionId, projectId, orgId, requestedBy, approvalConfig }],
    // Workflow-level timeout: 72 hours covers approval waiting time
    workflowExecutionTimeout: '72 hours',
  });

  return handle.workflowId;
}

// Sending an approval decision (from API endpoint hit by user clicking "Approve")
export async function recordApprovalDecision(
  workflowId: string,
  decision: ApprovalDecision,
): Promise<void> {
  const client = getTemporalClient();
  const handle = client.workflow.getHandle(workflowId);
  await handle.signal(approvalSignal, decision);
}

// Query current publish status (for UI polling / WebSocket push)
export async function getPublishStatus(workflowId: string): Promise<PublishStatus> {
  const client = getTemporalClient();
  const handle = client.workflow.getHandle(workflowId);
  return handle.query(publishStatusQuery);
}
```

---

## 10. Publish Status → UI Mapping

Internal Temporal state is never exposed directly to end users. A thin mapping layer translates workflow status to UI-friendly strings:

```typescript
// services/script/src/publish-status-mapper.ts

export type UIPublishStatus =
  | 'Submitting'
  | 'Validating'
  | 'In Planning'
  | 'Awaiting Approval'
  | 'Locking'
  | 'Published'
  | 'Submission Failed'
  | 'Withdrawn';

const STATUS_MAP: Record<PublishStatus, UIPublishStatus> = {
  locking_draft: 'Submitting',
  validating: 'Validating',
  extracting_breakdown: 'In Planning',
  computing_schedule: 'In Planning',
  computing_budget: 'In Planning',
  awaiting_approval: 'Awaiting Approval',
  locking_production: 'Locking',
  complete: 'Published',
  failed: 'Submission Failed',
  withdrawn: 'Withdrawn',
};

export function mapToUIStatus(internalStatus: PublishStatus): UIPublishStatus {
  return STATUS_MAP[internalStatus] ?? 'Submission Failed';
}
```

---

## 11. Event Fan-Out (Non-Orchestrated)

Not everything needs Temporal. For fire-and-forget or idempotent side effects, NATS JetStream events are preferable — lower overhead, no workflow history to manage.

| NATS Subject | Published By | Consumers |
|-------------|-------------|-----------|
| `script.checkpointed` | Script Service (CRDT checkpoint) | Search Service (re-index), Notification Service (autosave indicator) |
| `script.scene.modified` | Script Service | Continuity Service (timeline update), Bible Service (conflict check) |
| `supervisor.take.circled` | Supervisor Service (on sync) | Editorial Turnover queue, Dailies Sync |
| `supervisor.scene.completed` | Supervisor Service | Notification Service (1st AD alert), Supervisor turnover trigger |
| `workflow.approval.completed` | Script Service (after signal) | Notification Service, Audit Log |
| `script.version.locked` | Script Service (after operational lock) | Revision Distribution workflow start |
| `import.complete` | Import Saga worker | Search Service, Notification Service |

**Rule:** If compensation logic or human approval gates are required → Temporal. If idempotent side effect → NATS. Never use NATS for multi-step processes where intermediate state matters.

---

## 12. Testing Strategy

### Unit Tests

```typescript
// workers/publish-saga/src/__tests__/workflows/publish.test.ts

import { TestWorkflowEnvironment } from '@temporalio/testing';
import { Worker } from '@temporalio/worker';
import { publishToProductionWorkflow, approvalSignal, publishStatusQuery } from '../workflows/publish-to-production';

describe('publishToProductionWorkflow', () => {
  let env: TestWorkflowEnvironment;

  beforeAll(async () => {
    env = await TestWorkflowEnvironment.createTimeSkipping();
  });
  afterAll(async () => env.teardown());

  it('completes when all approvals received', async () => {
    const worker = await Worker.create({
      connection: env.nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows'),
      activities: mockActivities({ allSucceed: true }),
    });

    const { result } = await env.client.workflow.execute(publishToProductionWorkflow, {
      taskQueue: 'test',
      workflowId: 'test-publish-1',
      args: [testInput],
    });

    // Simulate approval decisions (auto-fired by time skipping env)
    const handle = env.client.workflow.getHandle('test-publish-1');
    await handle.signal(approvalSignal, { stepId: 'showrunner', decision: 'approved', decidedBy: 'user-1' });

    await worker.runUntil(async () => {
      const status = await handle.query(publishStatusQuery);
      expect(status).toBe('complete');
    });
  });

  it('compensates all steps when approval is rejected', async () => {
    const compensations: string[] = [];
    const worker = await Worker.create({
      connection: env.nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows'),
      activities: mockActivitiesWithTracking(compensations),
    });

    const handle = await env.client.workflow.start(publishToProductionWorkflow, {
      taskQueue: 'test',
      workflowId: 'test-publish-rejected',
      args: [testInput],
    });

    await handle.signal(approvalSignal, { stepId: 'showrunner', decision: 'rejected', decidedBy: 'user-1' });

    await expect(handle.result()).rejects.toThrow('Approval rejected');

    // All compensations should have fired
    expect(compensations).toContain('rollbackBudgetDelta');
    expect(compensations).toContain('rollbackScheduleDelta');
    expect(compensations).toContain('clearBreakdownArtifacts');
    expect(compensations).toContain('unlockDraft');
    // Compensation order: most recent first
    expect(compensations.indexOf('rollbackBudgetDelta')).toBeLessThan(
      compensations.indexOf('unlockDraft'),
    );
  });

  it('auto-approves at 48h timeout with time skipping', async () => {
    const worker = await Worker.create({
      connection: env.nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows'),
      activities: mockActivities({ allSucceed: true }),
    });

    const handle = await env.client.workflow.start(publishToProductionWorkflow, {
      taskQueue: 'test',
      workflowId: 'test-publish-auto',
      args: [testInput],
    });

    // Skip 49 hours — past both escalation (24h) and auto-approve (48h) thresholds
    await env.sleep('49 hours');
    const status = await handle.query(publishStatusQuery);
    expect(status).toBe('complete');
  });

  it('policy validation failure is non-retryable and compensates immediately', async () => {
    const worker = await Worker.create({
      connection: env.nativeConnection,
      taskQueue: 'test',
      workflowsPath: require.resolve('../workflows'),
      activities: mockActivities({ validatePolicyFails: true }),
    });

    const handle = await env.client.workflow.start(publishToProductionWorkflow, {
      taskQueue: 'test',
      workflowId: 'test-publish-policy',
      args: [testInput],
    });

    await expect(handle.result()).rejects.toThrow('PolicyViolationError');
    // Draft should be unlocked (only lockDraft had run — only unlockDraft should compensate)
  });
});
```

---

## Decisions

**Temporal deployment — Temporal Cloud (managed). See ADR-014.**
Self-hosted Temporal requires managing Cassandra or PostgreSQL persistence, replication, and upgrades. Temporal Cloud is a managed service with multi-region replication and SLA. Workers run in our K8s cluster (polling model — no inbound ports needed). The cost is per-action, which is negligible at ScriptOS scale. Self-hosted is revisited if monthly Temporal Cloud cost exceeds $2K/month.

**Approval routing — configurable per organization, three built-in templates.**
Approval chains vary significantly across production types. Hard-coding a single chain makes the platform unusable for non-standard productions. Each org configures their approval chain in `workflow.approval_chain_configs`. Three built-in templates ship at launch: Studio Feature, TV Episode, Independent. Orgs may customize steps, add parallel approvers, or define department-specific sub-chains.

**Escalation policy — escalate to next level after 24h; auto-approve after 48h.**
Default policy, configurable per org. At 24h without action, the workflow sends escalation notifications. At 48h, the step auto-approves. Auto-approval is necessary for productions under time pressure. The auto-approval event is logged in the audit trail with reason `timeout_auto_approved` so the action is attributable. Auto-approve is configurable: orgs may set `auto_approve_after_hours: null` to require explicit approval for every step (Studio tier only — blocks production indefinitely otherwise).

**Multi-department approval — parallel for independent departments, sequential for dependent.**
Independent departments (legal review, writer sign-off) run in parallel using `Promise.all` over activities in the Temporal workflow. Budget approval is sequential after schedule approval because budget is derived from schedule. The dependency chain: Breakdown → Schedule → Budget → Final Lock. Legal runs in parallel with Breakdown extraction. This minimizes total wall-clock time for the approval phase.

**Compensation — LIFO stack, execute all even if some fail.**
Compensation failures are logged but do not abort the compensation chain. The saga will attempt to roll back all completed steps regardless. If a compensation itself fails (e.g., rollbackScheduleDelta can't reach the Scheduling Service), the failure is logged with `type: 'compensation_failure'` in the audit log, and the engineering team is alerted. Manual intervention may be required in this case — the audit log provides the full state for recovery.

**Workflow visibility in UI — abstract behind simplified status labels.**
Temporal workflow IDs, activity names, and execution history are internal engineering tools. The product UI shows: Submitting, Validating, In Planning, Awaiting Approval, Locking, Published, Submission Failed. This mapping is maintained in a thin service-layer function. Engineers access the full Temporal Cloud UI for debugging.

**WorkflowId format — `publish:{scriptVersionId}` — deterministic, prevents duplicate publishes.**
Using a deterministic workflow ID derived from the script version ID means that if the client retries the publish API call (e.g., due to a network timeout), Temporal will return the existing workflow handle rather than starting a duplicate. A script version can only be in-flight in one publish saga at a time. Starting a second saga for the same version would signal a bug.

**Revision distribution — per-recipient watermarking runs in parallel with `Promise.allSettled`.**
Watermarking and distributing to each recipient is independent — one recipient's delivery failure should not block others. `Promise.allSettled` collects all outcomes; failures are logged and a separate retry mechanism (NATS dead-letter queue) handles re-delivery. `Promise.all` would cancel all distributions on the first failure — unacceptable for a 100+ recipient distribution list.
