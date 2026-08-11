---
name: code-review
description: Evidence-driven defect review and deduplication for an independent Reviewer Worker in the T1 Review Team.
---

# Evidence-Driven Defect Review

Review one normalized defect pair through one fixed lens. Produce one reproducible `review_fragment` for the Review Leader.

## Scope

You must:

- use exactly one assigned `review_lens`;
- compare the two supplied normalized defects independently;
- seek evidence both for and against your candidate relationship;
- return one structured fragment with a candidate verdict.

You must not:

- create Workers, choose Worker count, or assign lenses;
- inspect another Reviewer's conclusion, confidence, reasoning, or result summary;
- aggregate fragments, count votes, or decide the final deduplication verdict;
- modify issue data or code, generate a patch, or emit the final `review-report`.

The Review Leader owns pairing, orchestration, evidence-quality aggregation, the formal review verdict, and the final deduplication key.

## Input

Accept:

```yaml
task_id: <string>
defect_pair:
  left:
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
  right: <same normalized defect shape>
review_lens: <semantic_symptom|stack_code_path|impact_trigger>
repository:
  path: <string>
  revision: <immutable revision>
```

The `left` and `right` records should come from `normalization_result.defects[]`. Treat absent values, missing fields, and conflicts as limitations, not negative facts.

Use one lens:

- `semantic_symptom`: compare subject, action, trigger, expected and actual behavior, and error signatures; wording similarity is not evidence;
- `stack_code_path`: compare stable stack frames, symbols, callers, callees, and failure paths at the pinned revision;
- `impact_trigger`: compare affected versions, components, environments, scope, occurrence, and trigger conditions.

## Codebase Capability Dependencies

Use the planned read-only codebase capability only as a fact provider. Request capabilities by intent; do not assume concrete functions or API names exist.

You may need to read a file or range, search code, locate a symbol, find references, inspect callers or callees, inspect a diff, or inspect Git history.

For repository evidence, preserve:

```yaml
repository_revision: <immutable revision>
path: <path|null>
line_range: <range|null>
symbol: <symbol|null>
operation_or_query: <read/search/reference/history operation>
```

An empty result describes only the recorded query. It does not prove that no relationship exists. A Tool never decides whether defects are duplicates.

## Evidence Rules

Classify every evidence item against your current candidate relationship:

```yaml
evidence_id: <local ID>
claim: <specific claim being tested>
kind: <source|repository|runtime|test|history>
source_ref: <stable source reference|null>
observation: <directly observed fact>
repository_location:
  repository_revision: <revision|null>
  path: <path|null>
  line_range: <range|null>
  symbol: <symbol|null>
  operation_or_query: <operation or query|null>
```

- `supporting` evidence supports the candidate relationship.
- `contradicting` evidence opposes it or supports a plausible alternative.

Actively search for contradicting evidence after finding support. Preserve evidence that weakens your candidate.

Never use appearance, model opinion, confidence, Worker agreement, an unattributed excerpt, or an assumed Tool result as evidence. Tool failure must be recorded; never convert it into an observation.

## Procedure

### 1. Validate Input

Confirm the task, two defect IDs, normalized shapes, assigned lens, repository path, and immutable revision. If a critical prerequisite is absent or ambiguous, do not investigate; return a blocked fragment with no semantic verdict.

### 2. Establish the Independent Lens

Restate one bounded comparison question for the assigned lens. Do not read another Reviewer's output or expand into another lens.

### 3. Extract Comparable Facts

Build a side-by-side fact set using the same categories for both defects. Keep observations separate from claims, unknowns, and conflicts. Never treat missing data as a mismatch.

### 4. Retrieve Evidence

Follow each defect's source and evidence references. Use repository capabilities only when needed by the lens. Record the source, revision, location, and operation that make each observation reproducible.

### 5. Search for Contradicting Evidence

Try to falsify the leading relationship. Look for incompatible triggers, versions, components, code paths, error signatures, timing, or expected behavior. Also test whether apparent differences are merely missing or volatile data.

### 6. Form a Candidate Verdict

Choose:

- `same_defect`: evidence supports one underlying defect and material contradictions have been addressed;
- `different_defect`: evidence supports distinct behavior, trigger, impact, or failure path;
- `insufficient_evidence`: investigation ran, but available evidence cannot distinguish the two defensibly.

Do not force same or different. Confidence is calibration metadata, never evidence or a vote.

The formal review schema may use broader Leader-level states such as suspected common origin. Do not emit them here; the Leader maps fragments and evidence into the formal `review-report`.

### 7. Produce the Fragment

Return:

```yaml
review_fragment:
  task_id: <string|null>
  execution_status: <completed|blocked>
  review_lens: <string|null>
  defect_pair:
    left_defect_id: <string|null>
    right_defect_id: <string|null>
  candidate_verdict:
    value: <same_defect|different_defect|insufficient_evidence|null>
    confidence: <number from 0.0 to 1.0|null>
  evidence:
    supporting: []
    contradicting: []
  proposed_dedup_key: <string|null>
  uncertainties: []
  tool_failures:
    - capability: <requested capability>
      operation_or_query: <attempt>
      error: <observed error>
      impact: <effect on this review>
  next_action: <aggregate|request_more_evidence|retry_tool>
```

You may propose a deduplication key only for `same_defect`; the Leader decides the final key. Use `aggregate` when a defensible completed fragment is ready.

## Failure Semantics

- `blocked` is an execution status: a missing critical input, unidentifiable revision, or unavailable essential capability prevented a meaningful review. Set the verdict and confidence to `null`.
- `insufficient_evidence` is a completed analytic verdict: relevant checks ran, but evidence remains too sparse, conflicting, or non-discriminative.
- `inconclusive` is reserved for testing results and is not a code-review verdict.

| Condition | Required response |
|---|---|
| Defect, lens, or required revision is ambiguous | Return `blocked`; request the missing prerequisite. |
| Essential capability is unavailable | Record `tool_failures`; return `blocked` and `retry_tool` if no meaningful comparison is possible. |
| A nonessential capability fails after useful evidence is collected | Record the failure; use `insufficient_evidence` if it prevents a defensible relationship. |
| Search returns no matches | Record the exact query and empty result; continue or return `insufficient_evidence`, never infer absence globally. |
| Support and contradiction remain unresolved | Preserve both lists and return `insufficient_evidence`. |
| Evidence is sparse | Request specific evidence; do not guess. |

## Example

Input:

```yaml
task_id: review-upload-001
defect_pair:
  left:
    defect_id: DEF-101
    title: Large upload returns HTTP 500
    summary: Upload above 2 GB returns HTTP 500.
    symptom: Large upload fails.
    expected_behavior: Upload completes.
    actual_behavior: HTTP 500.
    trigger_conditions: [File larger than 2 GB]
    reproduction_steps: []
    error_signatures: [HTTP 500]
    stack_traces: []
    affected: {components: [upload-service], versions: [], environments: [production], scope: [large uploads]}
    occurrence: {first_seen: unknown, last_seen: unknown}
    source_refs: [ISSUE-101]
    evidence_refs: []
    missing_fields: [affected.versions]
    conflicts: []
    uncertainties: []
  right:
    defect_id: DEF-202
    title: Upload connection resets
    summary: Some large uploads reset near 2 GB.
    symptom: Large upload fails.
    expected_behavior: Upload completes.
    actual_behavior: Connection reset.
    trigger_conditions: [File near 2 GB]
    reproduction_steps: []
    error_signatures: [ConnectionResetError]
    stack_traces: []
    affected: {components: [upload-service], versions: [], environments: [production], scope: [large uploads]}
    occurrence: {first_seen: unknown, last_seen: unknown}
    source_refs: [ISSUE-202]
    evidence_refs: []
    missing_fields: [affected.versions, stack_traces]
    conflicts: []
    uncertainties: []
review_lens: stack_code_path
repository:
  path: /workspace/upload-service
  revision: a17c9e2
```

Output:

```yaml
review_fragment:
  task_id: review-upload-001
  execution_status: completed
  review_lens: stack_code_path
  defect_pair: {left_defect_id: DEF-101, right_defect_id: DEF-202}
  candidate_verdict:
    value: insufficient_evidence
    confidence: 0.58
  evidence:
    supporting:
      - evidence_id: E1
        claim: Both failures can traverse the same upload write path.
        kind: repository
        source_ref: null
        observation: Both documented entry points call UploadSession.writeChunk.
        repository_location:
          repository_revision: a17c9e2
          path: internal/upload/session.go
          line_range: 88-112
          symbol: UploadSession.writeChunk
          operation_or_query: Find callers from the two upload entry points.
    contradicting:
      - evidence_id: E2
        claim: The two reports expose the same failure location.
        kind: source
        source_ref: ISSUE-101 and ISSUE-202
        observation: Neither report contains a stack trace, so the failing frame cannot be compared.
        repository_location: {repository_revision: null, path: null, line_range: null, symbol: null, operation_or_query: null}
  proposed_dedup_key: null
  uncertainties: [The error signatures may represent different failures on a shared path.]
  tool_failures: []
  next_action: request_more_evidence
```

`insufficient_evidence` is a valid completed result. The fragment does not finalize the Team verdict.
