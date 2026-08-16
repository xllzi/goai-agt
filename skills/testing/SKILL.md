---
name: testing
description: Evidence-driven independent repair validation for a Tester Worker in the T3 Fix Team.
---

# Independent Repair Validation

Evaluate one `repair_candidate` through one fixed validation lens. Attack the candidate independently and return one reproducible `test_fragment` for the Fix Leader.

## Scope

You must:

- use exactly one assigned `test_lens`;
- derive explicit acceptance and rejection criteria from the defect, repair goal, and stable contracts;
- verify RED before GREEN when the lens and environment permit;
- inspect for new failures within the assigned scope;
- preserve revisions, commands, environments, exit codes, diagnostics, and failures;
- separate evidence supporting your result from evidence contradicting it.

You must not:

- create Testers, choose their count, or assign lenses;
- inspect another Tester's conclusion, confidence, private reasoning, or result summary;
- consume the Fixer's private reasoning or confidence;
- modify production code or alter the supplied patch;
- create a new fix plan, decide root cause, aggregate fragments, or vote;
- emit the final `fix-report`, accept the complete repair, deploy it, or announce production success.

The Fix Leader owns dispatch, cross-lens aggregation, formal repair review verdict, and final acceptance.

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
repair_candidate:
  selected_fix_candidate_id: <unchanged FixPlan selected_candidate_id>
  candidate_id: <stable ID>
  status: <ready>
  root_cause_ref: <stable reference>
  fix_plan_ref: <stable reference>
  repository:
    path: <string>
    base_revision: <immutable pre-repair revision>
    candidate_revision: <immutable commit, snapshot, or base-plus-patch identifier>
  repair_goal: <string>
  changes: []
  patch:
    diff: <inline diff|null>
    diff_ref: <stable reference|null>
    digest: <digest|null>
  affected_scope:
    direct_files: []
    symbols: []
    components: []
    callers: {check_status: <checked|not_checked|blocked>, findings: []}
    shared_state: {check_status: <checked|not_checked|blocked>, findings: []}
    configuration: {check_status: <checked|not_checked|blocked>, findings: []}
    timing_concurrency: {check_status: <checked|not_checked|blocked>, findings: []}
    compatibility: {check_status: <checked|not_checked|blocked>, findings: []}
    data_entities: []
    dependencies: []
  behavior:
    expected_change: <string>
    preserved_behavior: []
  risks: []
  assumptions: []
  validation_requests: []
test_lens: <string>
test_environment:
  runtime: <string>
  dependencies: <string|object>
  constraints: []
```

Consume the `repair_candidate` directly; do not translate it into a separate `fix_context`. Treat the task, normalized defect, unchanged selected FixPlan candidate ID, implementation candidate ID, formal references, exact before and candidate states, patch, bounded lens, runtime, and dependency context as required.

Common lenses are:

- `repair_review`: adversarially inspect causal mapping, diff scope, contracts, risks, and blast radius as an independent gate;
- `reproduction`: prove the original defect fails before and the same discriminative test passes after;
- `regression`: verify existing behavior adjacent to the changed module;
- `boundary`: test relevant null, empty, extreme, limit, exceptional, and boundary states;
- `integration`: verify affected module or dependency contracts;
- `impact_scope`: validate affected callers, shared state, configuration, data, dependencies, and timing.

Accept future Team-assigned lenses when they have one bounded objective. Use exactly one lens per invocation. Record uncovered areas; do not perform all lenses yourself.

The Fix Team should complete the independent `repair_review` gate before dispatching ordinary parallel test lenses. Worker creation, filtered visible inputs, execution order, and gate enforcement remain Team responsibilities; this Skill does not schedule or aggregate them.

## Isolation

Use only the normalized defect, formal root-cause and fix-plan references, final candidate, stable contracts, test environment, and raw Tool evidence.

Do not use the Fixer's justification narrative as an oracle. Do not use another Tester's verdict as evidence. Try to disprove the candidate's promised behavior and preservation claims.

## Capability Dependencies

Request capabilities by intent; do not assume concrete APIs or function names.

### Planned test-runner capability

You may need to discover existing tests, run a selected test or bounded suite, and capture exact command, suite type, repository revision, environment, exit code, stdout or stderr summary, passed and failed test names, duration, and timeout state.

### Planned static-analysis capability

You may need to execute an analyzer at the base and candidate states, capture rule, severity, repository revision, path, line, message, and analyzer configuration, and identify diagnostics introduced by the candidate.

When a static-analysis run contributes to formal verification, also record it as a real execution with `suite_type: static`, its exact command, repository revision, exit code, and result evidence. Diagnostics alone do not prove that an analyzer ran.

### Planned codebase capability

You may need read-only diff inspection, changed-file and changed-symbol confirmation, symbol lookup, references, callers, callees, history, and bounded impact-scope discovery.

Tools provide observations and execution mechanics. They do not determine whether the repair passes. Never fabricate an execution, diagnostic, code location, or Tool response.

## Evidence Rules

Classify every evidence item against your candidate validation result:

```yaml
evidence_id: <local ID>
claim: <acceptance or rejection criterion being tested>
kind: <test|static_analysis|repository|runtime|diff>
source_ref: <execution, diagnostic, or artifact reference>
observation: <directly observed fact>
repository_location:
  repository_revision: <base or candidate revision|null>
  path: <path|null>
  line_range: <range|null>
  symbol: <symbol|null>
  operation_or_query: <operation or query|null>
execution:
  suite_type: <unit|integration|boundary|mutation|static|e2e|null>
  command: <exact command|null>
  exit_code: <integer|null>
```

- `supporting` evidence supports your result value.
- `contradicting` evidence opposes it, reveals instability, or supports an alternative outcome.

For a `passed` result, successful relevant executions are supporting evidence and any suspicious signal belongs under contradicting evidence. For a `failed` result, the reproducible failure is supporting evidence; passing adjacent checks that narrow its scope may be contradicting evidence.

Never use appearance, prediction, confidence, Worker agreement, an unexecuted command, or a result without an attributable repository state as evidence. An empty test discovery result is not a pass.

## RED to GREEN

For a reproducible defect:

1. Run the discriminative test against `base_revision`.
2. Verify the expected defect-specific failure and record RED.
3. Run the same test with equivalent input and environment against `candidate_revision`.
4. Verify the expected behavior and record GREEN.

Do not run only the candidate state and claim the defect is fixed. An unrelated build error, dependency error, timeout, or different assertion is not verified RED.

If RED cannot be executed or does not reproduce the expected defect, set it to `not_verified`, explain why, and normally return `inconclusive`. For non-reproduction lenses, use `not_applicable` with a reason.

## Procedure

### 1. Validate the Candidate and Environment

Confirm `status: ready`, stable formal references, patch identity, repository path, base revision, candidate revision, lens, and environment. Confirm that the candidate revision represents the supplied patch exactly.

If a critical prerequisite is absent or ambiguous, do not execute against an assumed state; return `blocked`.

### 2. Establish One Independent Lens

Restate the lens as one bounded adversarial question. Do not inspect other Tester outputs or expand to another lens.

### 3. Derive Targets and Oracles

Define the behavior, input, relevant state, expected before and after results, explicit pass and fail criteria, and affected callers or contracts. Derive the oracle from reported expected behavior and stable contracts, not the Fixer's belief.

For `repair_review`, define the causal mapping, scope, contract, and risk claims to challenge.

For every planned real execution, assign exactly one suite type from the FixReport taxonomy:

- `unit`: an isolated unit or component-level executable test;
- `integration`: interaction between modules, services, dependencies, or fixtures;
- `boundary`: an executable suite specifically organized around limits or exceptional boundaries;
- `mutation`: a mutation-testing run;
- `static`: static analysis, lint, type checking, or another non-runtime analyzer;
- `e2e`: an end-to-end system or user-flow test.

`test_lens` and `suite_type` are independent. For example, reproduction may use an integration suite and regression may use a unit suite. Never use `reproduction`, `regression`, `impact_scope`, or `repair_review` as a suite type. If the execution cannot be classified from its harness and scope, obtain the missing metadata before running or reporting it; do not make the Fix Leader infer a type from the command.

### 4. Discover Existing Evidence and Tests

Prefer focused existing failing, unit, integration, and regression tests. Do not generate a large suite immediately. Record test discovery scope and queries.

If no relevant test exists, propose the smallest needed test under `uncertainties`. Do not claim it was created or executed.

### 5. Verify RED When Applicable

Run the selected reproduction against the base state. Preserve exact execution evidence. RED is verified only when the observed failure matches the defect.

### 6. Verify GREEN

Run the same reproduction against the candidate state. GREEN is verified only when the defect-specific assertion passes with an attributable successful execution.

### 7. Execute the Assigned Lens

- For `repair_review`, inspect the exact diff, causal mapping, callers, contracts, hidden scope, assumptions, and risks; produce evidence-linked findings.
- For `reproduction`, repeat only as needed to establish stability.
- For `regression`, run the smallest relevant existing suite around changed behavior.
- For `boundary`, test meaningful values below, at, and above the boundary plus relevant exceptional states.
- For `integration`, exercise the changed contract across affected boundaries.
- For `impact_scope`, validate evidenced callers, shared state, configuration, data, dependencies, or timing.

Record actual work only. Keep unexecuted proposals under `uncertainties`.

### 8. Check Differential Failures

Look for tests that passed before and fail after, new diagnostics, build changes, contract violations, changed caller behavior, performance regressions, timeouts, and nondeterminism within the lens.

One high-quality reproducible failure can justify `failed` regardless of how many checks passed.

### 9. Form the Candidate Result

Choose:

- `passed`: sufficient reproducible evidence satisfies this bounded lens and no blocking finding remains; reproduction requires verified RED and GREEN;
- `failed`: the defect remains, GREEN fails diagnostically, a reproducible regression appears, or a lens criterion is violated;
- `inconclusive`: meaningful validation ran, but results are conflicting, flaky, incomplete, or non-discriminative;
- `blocked`: a critical prerequisite, Tool, dependency, repository, build, or environment problem prevents meaningful validation.

`passed` applies only to this Tester and lens. Confidence is calibration metadata, not evidence or a vote.

### 10. Produce the Fragment

Return:

```yaml
test_fragment:
  task_id: <string|null>
  test_lens: <string|null>
  repair_reference:
    selected_fix_candidate_id: <unchanged FixPlan selected_candidate_id|null>
    candidate_id: <string|null>
    fix_plan_ref: <string|null>
    base_revision: <string|null>
    candidate_revision: <string|null>
    diff_ref: <string|null>
  result:
    value: <passed|failed|inconclusive|blocked>
    confidence: <number from 0.0 to 1.0>
  evidence:
    supporting: []
    contradicting: []
  red_green:
    red_phase:
      status: <verified_red|unexpected_pass|not_verified|not_applicable|blocked>
      evidence_refs: []
    green_phase:
      status: <verified_green|failed|not_verified|not_applicable|blocked>
      evidence_refs: []
  executions:
    - execution_id: <local ID>
      suite_type: <unit|integration|boundary|mutation|static|e2e>
      command: <exact command>
      repository_revision: <base or candidate revision>
      environment: <runtime and relevant dependencies>
      test_target: <test or suite>
      exit_code: <integer|null>
      passed_tests: []
      failed_tests: []
      failure_summary: <string|null>
      stdout_stderr_summary: <string|null>
      duration_ms: <integer|null>
      timed_out: <boolean>
  static_analysis:
    base_diagnostics: []
    candidate_diagnostics: []
    introduced_diagnostics: []
  review_findings:
    - severity: <blocking|high|medium|low>
      claim: <violated or uncertain criterion>
      evidence_refs: []
  regressions: []
  affected_scope_checked:
    files: []
    symbols: []
    components: []
    callers: []
    shared_state: []
    configuration: []
    timing_concurrency: []
    compatibility: []
    data_entities: []
    dependencies: []
  uncertainties: []
  tool_failures:
    - capability: <requested capability>
      operation_or_query: <attempt>
      error: <observed error>
      impact: <effect on validation>
  next_action: <aggregate|reject_candidate|request_more_validation|retry_tool>
```

Use `aggregate` for a defensible fragment, `reject_candidate` for a diagnostic failure, `request_more_validation` for inconclusive evidence, and `retry_tool` for a primary external blocker. Do not wrap the fragment as a final fix report.

## Failure Semantics

- `blocked`: validation could not meaningfully run because a critical prerequisite or external capability failed.
- `inconclusive`: meaningful validation ran, but its evidence cannot support pass or fail reliably.
- `insufficient_evidence`: reserved for review and investigation analytic verdicts; do not use it as a test result.

| Condition | Required response |
|---|---|
| Candidate state, environment, or dependency is unavailable | Record the limitation and return `blocked`. |
| Test-runner or another essential capability is unavailable | Record `tool_failures`; never simulate a run; return `blocked`. |
| Suite type cannot be established | Do not run or report an execution under an invented category; request harness or scope metadata and return `inconclusive` or `blocked` according to whether meaningful validation occurred. |
| Build fails at both states for an unrelated reason | Preserve both failures and return `blocked`. |
| Build passes before and fails after because of the candidate | Record a regression and return `failed`. |
| RED is not reproduced | Mark `not_verified` or `unexpected_pass`; return `inconclusive`. |
| GREEN fails diagnostically | Preserve the failure and return `failed`. |
| New regression or blocking review finding exists | Preserve evidence and return `failed`. |
| Results alternate between pass and fail | Preserve every run, mark flaky, return `inconclusive`, and never use majority voting. |
| Execution times out | Preserve command, duration, and partial output; return based on whether the timeout is infrastructure, nondiagnostic, or the tested failure itself. |
| No relevant tests are found | Record discovery evidence and return `inconclusive`; absence is not success. |

## Example

Input:

```yaml
task_id: test-upload-001
defect:
  defect_id: DEF-UPLOAD-2GB
  title: Upload fails above 2 GB
  summary: Large upload resets when byte accounting overflows.
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
repair_candidate:
  selected_fix_candidate_id: FIX_001
  candidate_id: RCAND-upload-001
  status: ready
  root_cause_ref: FIX-PLAN-9#RC-1
  fix_plan_ref: FIX-PLAN-9
  repository: {path: /workspace/upload-service, base_revision: a17c9e2, candidate_revision: "snapshot:a17c9e2+sha256:4a91"}
  repair_goal: Preserve byte accounting above 2 GB.
  changes: [{file: internal/upload/chunker.go, symbols: [Chunker.processedBytes]}]
  patch: {diff: null, diff_ref: patch.diff, digest: "sha256:4a91"}
  affected_scope:
    direct_files: [internal/upload/chunker.go]
    symbols: [Chunker.processedBytes]
    components: [upload-service]
    callers: {check_status: checked, findings: [UploadSession.writeChunk passes byte counts to the widened counter.]}
    shared_state: {check_status: checked, findings: [Chunker.processedBytes is the only changed shared state.]}
    configuration: {check_status: checked, findings: []}
    timing_concurrency: {check_status: checked, findings: [The type change adds no scheduling or locking behavior.]}
    compatibility: {check_status: checked, findings: [Public API types remain unchanged.]}
    data_entities: []
    dependencies: []
  behavior: {expected_change: Byte accounting works above 2 GB, preserved_behavior: [Small uploads remain unchanged.]}
  risks: [Caller conversion may truncate.]
  assumptions: []
  validation_requests: [{test_lens: reproduction, target: Original large upload, rationale: Establish RED to GREEN.}]
test_lens: reproduction
test_environment: {runtime: go1.24 linux-amd64, dependencies: sparse-file fixture, constraints: []}
```

Output:

```yaml
test_fragment:
  task_id: test-upload-001
  test_lens: reproduction
  repair_reference:
    selected_fix_candidate_id: FIX_001
    candidate_id: RCAND-upload-001
    fix_plan_ref: FIX-PLAN-9
    base_revision: a17c9e2
    candidate_revision: "snapshot:a17c9e2+sha256:4a91"
    diff_ref: patch.diff
  result: {value: passed, confidence: 0.94}
  evidence:
    supporting:
      - {evidence_id: T1, claim: Original defect is present before the candidate, kind: test, source_ref: run-red, observation: Test failed with the expected connection reset, repository_location: {repository_revision: a17c9e2, path: null, line_range: null, symbol: null, operation_or_query: Run reproduction.}, execution: {command: "go test ./internal/upload -run TestLargeUploadOver2GB -count=1", exit_code: 1}}
      - {evidence_id: T2, claim: Candidate prevents the original defect, kind: test, source_ref: run-green, observation: The same test completed with the full byte count, repository_location: {repository_revision: "snapshot:a17c9e2+sha256:4a91", path: null, line_range: null, symbol: null, operation_or_query: Run reproduction.}, execution: {command: "go test ./internal/upload -run TestLargeUploadOver2GB -count=1", exit_code: 0}}
    contradicting: []
  red_green:
    red_phase: {status: verified_red, evidence_refs: [T1]}
    green_phase: {status: verified_green, evidence_refs: [T2]}
  executions:
    - {execution_id: run-red, suite_type: integration, command: "go test ./internal/upload -run TestLargeUploadOver2GB -count=1", repository_revision: a17c9e2, environment: go1.24 linux-amd64, test_target: TestLargeUploadOver2GB, exit_code: 1, passed_tests: [], failed_tests: [TestLargeUploadOver2GB], failure_summary: connection reset at 2147483648 bytes, stdout_stderr_summary: expected defect-specific failure, duration_ms: 1840, timed_out: false}
    - {execution_id: run-green, suite_type: integration, command: "go test ./internal/upload -run TestLargeUploadOver2GB -count=1", repository_revision: "snapshot:a17c9e2+sha256:4a91", environment: go1.24 linux-amd64, test_target: TestLargeUploadOver2GB, exit_code: 0, passed_tests: [TestLargeUploadOver2GB], failed_tests: [], failure_summary: null, stdout_stderr_summary: pass, duration_ms: 1795, timed_out: false}
  static_analysis: {base_diagnostics: [], candidate_diagnostics: [], introduced_diagnostics: []}
  review_findings: []
  regressions: []
  affected_scope_checked: {files: [internal/upload/chunker.go], symbols: [Chunker.processedBytes], components: [upload-service], callers: [UploadSession.writeChunk], shared_state: [Chunker.processedBytes], configuration: [], timing_concurrency: [], compatibility: [Public API types unchanged], data_entities: [], dependencies: []}
  uncertainties: [Boundary, regression, and caller-conversion risks require separate Tester lenses.]
  tool_failures: []
  next_action: aggregate
```

This result validates only the reproduction lens. The Fix Leader must weigh it with independent repair-review and other test fragments.
