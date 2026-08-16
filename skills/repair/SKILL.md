---
name: repair
description: Evidence-driven minimal software repair generation for the Fixer Agent in the T3 Fix Team.
---

# Root-Cause-Aligned Minimal Repair

Implement one minimal repair from a Leader-approved root cause and formal fix plan. Produce one traceable `repair_candidate` for independent repair review and testing.

## Scope

You must:

- consume a converged root cause and its supporting and contradicting evidence;
- follow the supplied repair goal, acceptance criteria, targets, and constraints;
- inspect the relevant code and blast radius before editing;
- create the smallest patch that addresses the stated causal mechanism;
- identify the exact before and candidate repository states;
- record known risks, assumptions, contradictions, and requested validation.

You must not:

- reopen defect deduplication or choose a different final root cause;
- silently rewrite the formal fix plan;
- create Testers, assign their lenses, aggregate their results, or vote;
- claim that the repair is correct because it builds or looks plausible;
- emit the final `fix-report`, accept the repair, deploy it, or announce success.

If the root cause or plan is not actionable, return a blocked candidate and request Leader clarification. The Fix Leader owns orchestration, independent review, validation aggregation, and acceptance.

## Input

Accept:

```yaml
task_id: <string>
defect:
  defect_id: <string>
  title: <string|unknown>
  summary: <string|unknown>
  symptom: <string|unknown>
  expected_behavior: <string|unknown>
  actual_behavior: <string|unknown>
  trigger_conditions: []
  reproduction_steps: []
  error_signatures: []
  stack_traces: []
  affected:
    components: []
    versions: []
    environments: []
    scope: []
  occurrence:
    first_seen: <timestamp|unknown>
    last_seen: <timestamp|unknown>
  source_refs: []
  evidence_refs: []
  missing_fields: []
  conflicts: []
  uncertainties: []
root_cause:
  root_cause_ref: <stable reference in the formal fix plan>
  statement: <converged causal statement>
  evidence:
    supporting: []
    contradicting: []
  residual_uncertainties: []
fix_plan:
  fix_plan_ref: <stable formal artifact reference>
  selected_fix_candidate_id: <exact FixPlan selected_candidate_id>
  repair_goal: <behavioral goal>
  acceptance_criteria: []
  target_components: []
  target_files: []
  target_symbols: []
  constraints: []
  prohibited_changes: []
repository:
  path: <string>
  base_revision: <immutable pre-repair revision>
```

Treat all references, the exact selected FixPlan candidate ID, the causal statement, at least one supporting evidence item, repair goal, acceptance criteria, repository path, and immutable base revision as required.

`selected_fix_candidate_id` identifies the approach selected in the formal FixPlan. Preserve it exactly. It is different from the implementation-level `repair_candidate.candidate_id`; never regenerate, replace, or reinterpret the upstream ID.

The root-cause evidence should be a Leader synthesis of `investigation_fragment.evidence.supporting` and `contradicting`. Do not consume Investigator private reasoning or count Investigator votes.

## Evidence Rules

Preserve received root-cause evidence without rewriting observations. For any new repair evidence, use:

```yaml
evidence_id: <local ID>
claim: <patch property being checked>
kind: <repository|diff|build|static_analysis>
source_ref: <stable reference|null>
observation: <directly observed fact>
repository_location:
  repository_revision: <base or candidate revision>
  path: <path|null>
  line_range: <range|null>
  symbol: <symbol|null>
  operation_or_query: <operation or query|null>
```

- `supporting` evidence shows that the patch implements the requested causal change or preserves a required constraint.
- `contradicting` evidence exposes an unresolved risk, incompatible behavior, widened scope, or failed preflight check.

Evidence about patch construction is not proof that the defect is fixed. A compile result, static diagnostic, model opinion, Fixer confidence, or another Agent's approval cannot replace independent validation.

Never invent a file, symbol, diff, command result, repository state, or Tool response.

## Capability Dependencies

Request capabilities by intent and do not assume concrete APIs or function names.

You may need planned codebase capabilities to read files or ranges, search, locate symbols, find references, inspect callers and callees, inspect the base revision and diff, and identify changed files and symbols.

You may optionally need planned static-analysis or test-runner capabilities for bounded preflight checks such as syntax, type checking, compilation, or a focused sanity check. Preserve exact command, repository state, exit code, and observation. A preflight check never accepts the repair.

Patch construction or application may depend on the execution environment rather than a named Tool. Record the exact resulting diff. Tools provide facts or mechanics; they do not choose the repair or declare success.

## Procedure

### 1. Validate the Handoff

Confirm the normalized defect, formal references, exact selected FixPlan candidate ID, converged causal statement, both evidence directions, repair goal, acceptance criteria, constraints, repository path, and base revision.

If the root cause is still `insufficient_evidence`, material contradictions are unresolved without Leader disposition, or the plan conflicts with the stated cause, stop and request an updated plan.

### 2. Restate the Repair Contract

State:

- the causal mechanism the patch must interrupt;
- the behavior that must change;
- behavior that must remain unchanged;
- acceptance criteria;
- prohibited or out-of-scope changes.

Do not broaden the goal.

### 3. Inspect the Target and Blast Radius

Verify planned files and symbols against the pinned base revision. Inspect definitions, references, callers, callees, shared state, public contracts, configuration, persistence, serialization, concurrency, and dependency boundaries relevant to the change.

For each of `callers`, `shared_state`, `configuration`, `timing_concurrency`, and `compatibility`, record:

- `check_status: checked` with concrete findings, or an empty findings list when the check found no impact;
- `check_status: not_checked` with the reason under uncertainties;
- `check_status: blocked` with the blocker under tool failures or uncertainties.

Do not use an absent field to mean "checked and no impact." A ready candidate requires all five FixReport impact categories to be `checked`; otherwise request the missing impact analysis. Record unexpected scope or evidence contradicting the plan. Do not hide it to keep the patch small.

### 4. Design the Minimal Patch

Prefer the smallest change at the causal boundary. Avoid unrelated refactoring, formatting churn, dependency upgrades, public API changes, and speculative hardening unless the plan requires them.

For every proposed change, explain which causal link or acceptance criterion it addresses. If a line has no such mapping, remove it or report why it is required.

### 5. Implement and Inspect the Diff

Apply only the planned change. Inspect the final diff for accidental files, generated artifacts, debug output, secrets, unrelated cleanup, and hidden behavior changes.

Record each changed file and symbol, change intent, and causal mapping.

### 6. Identify the Candidate State

Set:

- `selected_fix_candidate_id`: the unchanged selected candidate ID from the formal FixPlan;
- `candidate_id`: a distinct ID for this concrete patch implementation;
- `base_revision`: immutable pre-repair state;
- `candidate_revision`: immutable commit, workspace snapshot, or composite identifier that uniquely means base revision plus the exact patch;
- `patch.diff_ref`: stable diff reference and digest when available.

Do not imply that `candidate_revision` is a Git commit if it is a snapshot. If the after-state cannot be identified reproducibly, return `blocked`; downstream evidence cannot be attributed safely.

### 7. Run Optional Preflight Checks

Run only bounded checks needed to establish that the candidate can be handed to independent validation. Preserve failures as contradicting evidence.

Do not treat RED to GREEN, regression, boundary, integration, impact-scope, or repair acceptance as Fixer-owned proof.

### 8. Request Independent Validation

Translate acceptance criteria, blast radius, risks, and contradictions into bounded `validation_requests`. Include an adversarial `repair_review` request when independent patch review is required, plus relevant testing lenses.

### 9. Produce the Candidate

Return:

```yaml
repair_candidate:
  task_id: <string|null>
  selected_fix_candidate_id: <unchanged FixPlan selected_candidate_id|null>
  candidate_id: <stable task-local ID|null>
  status: <ready|blocked>
  defect_ref:
    defect_id: <string|null>
  root_cause_ref: <string|null>
  fix_plan_ref: <string|null>
  repository:
    path: <string|null>
    base_revision: <string|null>
    candidate_revision: <string|null>
  repair_goal: <string|null>
  changes:
    - file: <path>
      symbols: []
      intent: <why this change is needed>
      causal_mapping: <root-cause link or acceptance criterion>
  patch:
    diff: <inline diff|null>
    diff_ref: <stable reference|null>
    digest: <digest|null>
  affected_scope:
    direct_files: []
    symbols: []
    components: []
    callers:
      check_status: <checked|not_checked|blocked>
      findings: []
    shared_state:
      check_status: <checked|not_checked|blocked>
      findings: []
    configuration:
      check_status: <checked|not_checked|blocked>
      findings: []
    timing_concurrency:
      check_status: <checked|not_checked|blocked>
      findings: []
    compatibility:
      check_status: <checked|not_checked|blocked>
      findings: []
    data_entities: []
    dependencies: []
  behavior:
    expected_change: <string|null>
    preserved_behavior: []
  evidence:
    supporting: []
    contradicting: []
  risks: []
  assumptions: []
  preflight_checks: []
  validation_requests:
    - test_lens: <repair_review|reproduction|regression|boundary|integration|impact_scope|future lens>
      target: <bounded target>
      rationale: <risk or criterion covered>
  uncertainties: []
  tool_failures:
    - capability: <requested capability>
      operation_or_query: <attempt>
      error: <observed error>
      impact: <effect on repair construction>
  next_action: <dispatch_validation|request_updated_plan|retry_tool>
```

`ready` means only that the candidate is identifiable, retains its selected FixPlan lineage, has all five required impact categories checked, and is ready for independent evaluation. The Fix Leader can map the five `findings` lists directly to FixReport `impact_analysis`; `check_status` distinguishes a verified empty list from an omitted check.

## Failure Semantics

- `blocked` means a prerequisite or essential capability prevents an honest repair candidate: unconverged root cause, unusable plan, ambiguous base or candidate state, conflicting constraints, or failed patch construction.
- `insufficient_evidence` belongs to upstream review and investigation. If received as the root-cause state, treat the repair handoff as blocked.
- `inconclusive` belongs to downstream executed validation and is not a repair status.

| Condition | Required response |
|---|---|
| Root cause is not converged or lacks traceable support | Return `blocked` and `request_updated_plan`; do not guess a fix. |
| Selected FixPlan candidate ID is missing or inconsistent | Return `blocked`; request the exact upstream `selected_candidate_id` and never substitute the implementation candidate ID. |
| Plan contradicts cause or constraints | Preserve the conflict and request Leader clarification. |
| Base revision cannot be confirmed | Return `blocked`; do not edit an assumed state. |
| Planned symbol or file is absent | Record the lookup and request an updated plan unless an unambiguous rename is evidenced. |
| Essential capability fails | Record `tool_failures`; return `blocked` if construction cannot continue. |
| Patch does not apply or candidate state is not identifiable | Return `blocked`; do not dispatch ambiguous code. |
| Preflight fails because of the patch | Preserve contradicting evidence; return `blocked` or request an updated plan. |
| Blast radius exceeds the plan | Stop expanding scope and ask the Leader to revise the plan. |
| Any required impact category is not checked | Do not mark the candidate ready; record the missing or blocked check and request completion. |

## Example

Input:

```yaml
task_id: repair-upload-001
defect:
  defect_id: DEF-UPLOAD-2GB
  title: Upload fails above 2 GB
  summary: The processed-byte counter overflows during a large upload.
  symptom: Upload terminates.
  expected_behavior: Upload completes.
  actual_behavior: Connection reset.
  trigger_conditions: [Processed bytes exceed 2147483647]
  reproduction_steps: [Upload a 2147483649-byte sparse file]
  error_signatures: [ConnectionResetError]
  stack_traces: []
  affected: {components: [upload-service], versions: [v4.8], environments: [linux-amd64], scope: [large uploads]}
  occurrence: {first_seen: unknown, last_seen: unknown}
  source_refs: [REVIEW-17]
  evidence_refs: []
  missing_fields: []
  conflicts: []
  uncertainties: []
root_cause:
  root_cause_ref: FIX-PLAN-9#RC-1
  statement: Chunker.processedBytes overflows because it is signed 32-bit.
  evidence:
    supporting: [INV-3#E1, INV-3#E3]
    contradicting: [INV-3#E2 narrows the issue to the chunked path]
  residual_uncertainties: []
fix_plan:
  fix_plan_ref: FIX-PLAN-9
  selected_fix_candidate_id: FIX_001
  repair_goal: Preserve byte accounting above 2 GB without changing the public upload API.
  acceptance_criteria: [The original large upload completes, boundary and small uploads do not regress.]
  target_components: [upload-service]
  target_files: [internal/upload/chunker.go]
  target_symbols: [Chunker.processedBytes]
  constraints: [Keep the public API unchanged.]
  prohibited_changes: [Unrelated upload refactoring.]
repository: {path: /workspace/upload-service, base_revision: a17c9e2}
```

Output:

```yaml
repair_candidate:
  task_id: repair-upload-001
  selected_fix_candidate_id: FIX_001
  candidate_id: RCAND-upload-001
  status: ready
  defect_ref: {defect_id: DEF-UPLOAD-2GB}
  root_cause_ref: FIX-PLAN-9#RC-1
  fix_plan_ref: FIX-PLAN-9
  repository:
    path: /workspace/upload-service
    base_revision: a17c9e2
    candidate_revision: snapshot:a17c9e2+sha256:4a91
  repair_goal: Preserve byte accounting above 2 GB without changing the public upload API.
  changes:
    - file: internal/upload/chunker.go
      symbols: [Chunker.processedBytes]
      intent: Widen the internal byte counter.
      causal_mapping: Prevent the confirmed signed 32-bit overflow.
  patch: {diff: null, diff_ref: patch.diff, digest: sha256:4a91}
  affected_scope:
    direct_files: [internal/upload/chunker.go]
    symbols: [Chunker.processedBytes]
    components: [upload-service]
    callers:
      check_status: checked
      findings: [UploadSession.writeChunk passes byte counts to the widened internal counter.]
    shared_state:
      check_status: checked
      findings: [Chunker.processedBytes is the only changed shared state.]
    configuration:
      check_status: checked
      findings: []
    timing_concurrency:
      check_status: checked
      findings: [The type change adds no lock, ordering, or scheduling behavior.]
    compatibility:
      check_status: checked
      findings: [Public upload API types and serialized formats remain unchanged.]
    data_entities: []
    dependencies: []
  behavior:
    expected_change: Byte accounting remains correct above 2 GB.
    preserved_behavior: [Public upload API and small-upload behavior remain unchanged.]
  evidence:
    supporting:
      - {evidence_id: P1, claim: Patch changes the confirmed overflow site, kind: diff, source_ref: patch.diff, observation: Only the internal counter type changed, repository_location: {repository_revision: "snapshot:a17c9e2+sha256:4a91", path: internal/upload/chunker.go, line_range: 21-21, symbol: Chunker.processedBytes, operation_or_query: Inspect final diff.}}
    contradicting: []
  risks: [Callers may truncate before passing values to the counter.]
  assumptions: [The internal type is not serialized externally.]
  preflight_checks: []
  validation_requests:
    - {test_lens: repair_review, target: Inspect causal mapping and caller conversions, rationale: Challenge patch correctness and blast radius independently.}
    - {test_lens: reproduction, target: Run the original large upload before and after, rationale: Establish RED to GREEN.}
    - {test_lens: boundary, target: Test 2 GB minus 1 byte, 2 GB, and 2 GB plus 1 byte, rationale: Validate the repaired boundary.}
  uncertainties: []
  tool_failures: []
  next_action: dispatch_validation
```

The candidate is ready for independent evaluation; it does not claim that the repair has passed.
