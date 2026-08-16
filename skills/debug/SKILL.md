---
name: debug
description: Evidence-driven single-hypothesis root cause investigation for an independent Investigator Worker in the T2 Investigation Team.
---

# Single-Hypothesis Investigation

Investigate one assigned root-cause hypothesis for one reviewed defect. Try to falsify it, preserve reproducible evidence, and return one `investigation_fragment` for the Investigation Leader.

## Scope

You must:

- investigate exactly one assigned hypothesis;
- derive observations that would support and reject it;
- actively seek falsifying evidence and record every falsification attempt;
- keep code, runtime, configuration, data, dependency, and test evidence traceable;
- return one candidate assessment.

You must not:

- create Investigators, select their count, or assign hypotheses;
- switch to or combine multiple hypotheses in one invocation;
- inspect another Investigator's conclusion, confidence, private reasoning, or result summary;
- aggregate fragments, vote, declare the final root cause, or generate the formal `fix-plan`;
- modify production code or generate a repair patch.

The Investigation Team owns hypothesis decomposition and parallel dispatch. The Leader weighs evidence across fragments, decides whether root cause has converged, and creates the formal fix plan.

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
review_context:
  review_report_ref: <stable reference>
  reviewed_defect_ref: <stable reference>
root_cause_hypothesis:
  hypothesis_id: <string>
  hypothesis_type: <code|configuration|data|dependency|environment|timing|other>
  statement: <one falsifiable causal statement>
  predicted_observations: []
  rejection_criteria: []
repository:
  path: <string>
  revision: <immutable revision>
investigation_context:
  environment: <string|object|null>
  constraints: []
```

Treat the normalized defect, formal review reference, one falsifiable hypothesis, repository path, and immutable revision as required. A hypothesis such as "the code is wrong" is not bounded enough; request a causal statement with observable predictions.

## Isolation

Use the reviewed defect, formal handoff facts, assigned hypothesis, pinned repository, approved environment facts, and raw Tool evidence. Do not consume:

- another Investigator's verdict, confidence, reasoning, or summary;
- the Leader's preferred hypothesis;
- a proposed fix as proof of causality.

If you discover a different plausible cause, record it under `alternative_hypotheses` for future Team dispatch. Do not investigate it in this fragment.

## Evidence Rules

Classify evidence against the assigned hypothesis:

```yaml
evidence_id: <local ID>
claim: <prediction or rejection criterion being tested>
kind: <repository|runtime|configuration|data|dependency|test|history>
source_ref: <stable source or artifact reference|null>
observation: <directly observed fact>
repository_location:
  repository_revision: <revision|null>
  path: <path|null>
  line_range: <range|null>
  symbol: <symbol|null>
  operation_or_query: <operation or query|null>
runtime_location:
  timestamp_or_version: <value|null>
  location_or_identifier: <trace, request, config, record, package, or environment identifier|null>
```

- `supporting` evidence increases the assigned hypothesis's explanatory power.
- `contradicting` evidence falsifies a prediction, satisfies a rejection criterion, or supports an incompatible explanation.

Seek contradicting evidence even after strong support. Correlation, plausibility, model opinion, Worker agreement, and confidence are not proof. Never invent a Tool result, runtime event, configuration value, data record, or execution.

Keep actual contradicting observations separate from the process used to search for them:

```yaml
falsification:
  status: <performed|not_performed|blocked>
  attempts:
    - attempt_id: <local ID>
      what_was_tested: <prediction or rejection criterion>
      expected_if_hypothesis_false: <observable counterexample>
      operation_or_query: <executed search, inspection, or experiment>
      observed_result: <direct observation>
      outcome: <contradicted|did_not_contradict|inconclusive|blocked>
      evidence_ref: <evidence ID|null>
  findings:
    - statement: <actual counter-evidence or scope-limiting finding>
      evidence_refs: []
```

`performed` requires at least one completed attempt. `not_performed` means no falsification attempt ran. `blocked` means every planned attempt was prevented by a recorded blocker. Keep `findings` empty when no genuine contradicting observation was found; never add a placeholder finding.

The Investigation Leader can map each genuine finding to a FixPlan `counter_evidence` string. When `status: performed` and `findings` is empty, the Leader can instead summarize a recorded attempt as: what was tested, the expected counterexample, the observed result, and that the result did not contradict the hypothesis. This is an auditable falsification result, not fabricated counter-evidence.

## Capability Dependencies

Request capabilities by intent; do not assume concrete APIs or function names.

### Planned codebase capability

You may need to read files or ranges, search code, locate symbols, inspect references, callers, callees, diffs, blame, or history. Repository evidence must carry revision, path, line range, symbol, and operation or query when applicable.

### Planned static-analysis capability

You may need to execute an analyzer and capture rule, severity, repository revision, path, line, message, and configuration.

### Planned test-runner capability

You may need to discover or run a focused reproduction and capture command, repository revision, environment, exit code, observed result, failure summary, and duration.

### Other fact sources

Configuration, logs, data state, dependency metadata, deployment facts, and monitoring signals may require future or external evidence providers. Describe the needed capability and locator without inventing a Tool name. If unavailable, record the limitation and do not substitute an assumption.

Tools return facts. They do not assess the hypothesis or decide root cause.

## Procedure

### 1. Validate the Task

Confirm the reviewed defect, formal review reference, one hypothesis, repository, revision, and constraints. If a critical prerequisite is absent or the hypothesis cannot be tested, return a blocked fragment without an assessment.

### 2. State Predictions and Rejection Criteria

Rewrite the assigned statement as one bounded causal question. List:

- observations expected if it is true;
- observations expected if it is false;
- at least one practical rejection criterion;
- the smallest evidence set that can discriminate between them.

Do not change the hypothesis to fit early evidence.

### 3. Plan Bounded Checks

Choose checks specific to the hypothesis type. Prefer the cheapest discriminative observations first. Pin every repository and environment state.

### 4. Gather Supporting Evidence

Retrieve only facts relevant to the predictions. Trace the observed symptom to the proposed mechanism instead of stopping at suspicious code or correlated timing.

### 5. Try to Falsify

Execute the rejection checks. Look for unaffected counterexamples, incompatible code paths, normal values under the failure condition, version mismatches, or reproductions that fail differently.

Record every planned and executed check under `falsification.attempts`. Set its outcome from the observation, link its evidence, and put only genuine counter-evidence under both `falsification.findings` and `evidence.contradicting`.

### 6. Define and Execute Reproduction or Verification

Define at least one concrete reproduction or verification step before completing the investigation. The step may have been executed, remain executable but not yet run, or be blocked by a named prerequisite.

Use:

- `executed` when the listed steps ran and have execution evidence;
- `not_executed` when the steps are executable but were not run;
- `blocked` when the steps are defined but a recorded environment, dependency, data, or Tool blocker prevented execution.

Preserve environment, preconditions, non-empty steps, expected result, actual result or explicit not-observed reason, command, revision, exit code, and failure summary. An unrelated failure is not support. If you cannot define even one meaningful step, do not complete a supported assessment.

### 7. Assess the Hypothesis

Choose:

- `supported`: multiple traceable observations connect the hypothesis to the symptom, falsification was performed, at least one reproduction or verification step is present, and rejection attempts do not reveal an unresolved material contradiction;
- `rejected`: reproducible contradicting evidence falsifies a required prediction or causal link;
- `insufficient_evidence`: relevant investigation ran, but evidence cannot support or reject the hypothesis defensibly.

Do not declare the final root cause. Confidence is calibration metadata, not evidence or a vote.

### 8. Produce the Fragment

Return:

```yaml
investigation_fragment:
  task_id: <string|null>
  execution_status: <completed|blocked>
  defect_ref:
    defect_id: <string|null>
    review_report_ref: <string|null>
  hypothesis:
    hypothesis_id: <string|null>
    hypothesis_type: <string|null>
    statement: <string|null>
  assessment:
    value: <supported|rejected|insufficient_evidence|null>
    confidence: <number from 0.0 to 1.0|null>
  evidence:
    supporting: []
    contradicting: []
  falsification:
    status: <performed|not_performed|blocked>
    attempts:
      - attempt_id: <local ID>
        what_was_tested: <string>
        expected_if_hypothesis_false: <string>
        operation_or_query: <string>
        observed_result: <string>
        outcome: <contradicted|did_not_contradict|inconclusive|blocked>
        evidence_ref: <string|null>
    findings:
      - statement: <actual counter-evidence or scope-limiting finding>
        evidence_refs: []
  reproduction:
    status: <executed|not_executed|blocked>
    environment: <string|object>
    preconditions: []
    steps: [<at least one executable step for a completed fragment>]
    expected: <string>
    actual: <observed result or explicit not-observed reason>
    executions: []
  affected_scope:
    files: []
    symbols: []
    components: []
    configurations: []
    data_entities: []
    dependencies: []
  alternative_hypotheses: []
  uncertainties: []
  tool_failures:
    - capability: <requested capability>
      operation_or_query: <attempt>
      error: <observed error>
      impact: <effect on this investigation>
  repair_hint:
    target: <component, file, symbol, config, data, or dependency|null>
    constraint: <what a later fix must preserve|null>
  next_action: <aggregate|request_more_evidence|retry_tool>
```

`repair_hint` is a bounded handoff suggestion, not a fix plan or patch. The Leader may accept, reject, or combine it only after evidence-quality aggregation.

## Failure Semantics

- `blocked` means a critical prerequisite or essential external capability prevented meaningful investigation. Set assessment and confidence to `null`.
- `insufficient_evidence` means relevant checks were performed but could not resolve the assigned hypothesis.
- `inconclusive` is reserved for executed repair validation and is not a debug assessment.

Never output `supported` unless `falsification.status: performed`, `falsification.attempts` is non-empty, `reproduction.steps` is non-empty, and supporting evidence is sufficient. If falsification was not performed or no executable verification step can be defined, use `insufficient_evidence` for a completed investigation or `blocked` when a critical prerequisite prevented investigation.

| Condition | Required response |
|---|---|
| Hypothesis is missing, compound, or unfalsifiable | Return `blocked` and request one testable statement. |
| Revision or required environment is unidentifiable | Return `blocked`; do not attribute evidence to an assumed state. |
| Essential capability is unavailable | Record `tool_failures`; return `blocked` and `retry_tool` if no meaningful check can run. |
| No falsification attempt was performed | Do not output `supported`; return `insufficient_evidence` and request a discriminative check. |
| Falsification ran but found no contradiction | Keep contradicting evidence and findings empty; preserve non-empty attempts and their observed outcomes. |
| No reproduction or verification step can be defined | Do not output `supported`; request the missing behavior, environment, or oracle. |
| Steps are defined but cannot run | Use reproduction `blocked`, preserve the steps and blocker, and assess only from other admissible evidence. |
| Search or analysis returns no result | Preserve the query and empty result; treat it as bounded evidence only. |
| Support and contradiction remain unresolved | Preserve both and return `insufficient_evidence`. |
| Reproduction fails for unrelated infrastructure reasons | Record the failure; do not count it as support or rejection. |
| A new hypothesis appears | Record it for Team reassignment; do not switch hypotheses. |

## Example

Input:

```yaml
task_id: investigate-upload-001
defect:
  defect_id: DEF-UPLOAD-2GB
  title: Upload fails above 2 GB
  summary: The connection resets when a large upload crosses 2 GB.
  symptom: Upload terminates.
  expected_behavior: Upload completes.
  actual_behavior: ConnectionResetError.
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
review_context:
  review_report_ref: REVIEW-17
  reviewed_defect_ref: REVIEW-17#DEF-UPLOAD-2GB
root_cause_hypothesis:
  hypothesis_id: H-CODE-1
  hypothesis_type: code
  statement: A signed 32-bit processed-byte counter overflows above 2 GB and terminates the upload path.
  predicted_observations: [The counter type is signed 32-bit, failure begins when it overflows.]
  rejection_criteria: [The counter remains correct above 2 GB or the failing path does not use it.]
repository: {path: /workspace/upload-service, revision: a17c9e2}
investigation_context: {environment: linux-amd64 integration fixture, constraints: []}
```

Output:

```yaml
investigation_fragment:
  task_id: investigate-upload-001
  execution_status: completed
  defect_ref: {defect_id: DEF-UPLOAD-2GB, review_report_ref: REVIEW-17}
  hypothesis:
    hypothesis_id: H-CODE-1
    hypothesis_type: code
    statement: A signed 32-bit processed-byte counter overflows above 2 GB and terminates the upload path.
  assessment: {value: supported, confidence: 0.93}
  evidence:
    supporting:
      - evidence_id: E1
        claim: The failing path uses a signed 32-bit byte counter.
        kind: repository
        source_ref: null
        observation: Chunker.processedBytes is int32 and is incremented by each chunk.
        repository_location: {repository_revision: a17c9e2, path: internal/upload/chunker.go, line_range: 21-47, symbol: Chunker.processedBytes, operation_or_query: Read declaration and update sites.}
        runtime_location: {timestamp_or_version: null, location_or_identifier: null}
    contradicting:
      - evidence_id: E2
        claim: Every large upload must fail at the same boundary.
        kind: test
        source_ref: run-44
        observation: A multipart path completed above 2 GB because it bypasses Chunker.
        repository_location: {repository_revision: a17c9e2, path: null, line_range: null, symbol: null, operation_or_query: Run multipart counterexample.}
        runtime_location: {timestamp_or_version: v4.8, location_or_identifier: linux-amd64}
  falsification:
    status: performed
    attempts:
      - attempt_id: F1
        what_was_tested: Whether the reported failing path can complete above 2 GB without using the signed counter.
        expected_if_hypothesis_false: The same chunked path completes above 2 GB or does not read Chunker.processedBytes.
        operation_or_query: Trace the failing entry point and run the large-upload reproduction at a17c9e2.
        observed_result: The chunked path reads the counter and it becomes negative; a separate multipart path bypasses it.
        outcome: did_not_contradict
        evidence_ref: E1
    findings:
      - statement: The multipart path completes above 2 GB because it bypasses Chunker, limiting the hypothesis to the chunked path.
        evidence_refs: [E2]
  reproduction:
    status: executed
    environment: linux-amd64 integration fixture
    preconditions: [Sparse-file fixture supports a 2147483649-byte upload.]
    steps:
      - Upload a 2147483649-byte sparse file through the chunked upload path.
      - Observe the processed-byte counter and connection result.
    expected: The upload completes and preserves the full byte count.
    actual: The counter became negative at 2147483648 bytes and the connection reset.
    executions:
      - {command: go test ./internal/upload -run TestLargeUploadOverflow, repository_revision: a17c9e2, exit_code: 1, observed_result: counter became negative at 2147483648 bytes}
  affected_scope:
    files: [internal/upload/chunker.go]
    symbols: [Chunker.processedBytes]
    components: [upload-service]
    configurations: []
    data_entities: []
    dependencies: []
  alternative_hypotheses: []
  uncertainties: [The multipart path is outside this causal mechanism.]
  tool_failures: []
  repair_hint: {target: Chunker.processedBytes, constraint: Preserve public upload API types and multipart behavior.}
  next_action: aggregate
```

The contradiction narrows scope rather than being hidden. Only the Leader may promote the hypothesis into a converged root cause and fix plan.
