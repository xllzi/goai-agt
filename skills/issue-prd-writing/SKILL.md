---
name: issue-prd-writing
description: Normalize heterogeneous software defect inputs into traceable structured defect records for downstream review and deduplication.
---

# Traceable Defect Normalization

Convert one or more raw defect sources into traceable normalized defects. Preserve source meaning, gaps, conflicts, and raw references so downstream Reviewers can compare facts without reinterpreting the original material.

## Scope

You must:

- identify each source and preserve its traceability metadata;
- extract observable facts separately from reporter claims;
- normalize field names and shapes conservatively;
- keep field-level evidence references, missing fields, conflicts, and uncertainties;
- return one or more normalized defects.

You must not:

- compare defects or decide whether they are duplicates;
- infer or verify a root cause;
- inspect source code, run tests, modify code, or create a patch;
- create Workers, assign work, aggregate Worker results, or emit a final review report.

The Review Team decides which normalized defects to compare. A Reviewer using the code-review Skill performs one comparison.

## Input

Accept:

```yaml
task_id: <string>
raw_sources:
  - source_type: <string>
    source_id: <string|null>
    source_uri: <string|null>
    timestamp: <string|null>
    reporter: <string|null>
    content: <string|object>
    metadata:
      version: <string|null>
      environment: <string|null>
      component: <string|null>
```

Common source types include issue, ticket, user feedback, log, incident, and test failure. Accept future types. Use `unknown_source` when classification is not reliable.

Group sources into one defect only when the task groups them explicitly or they contain a reproducible relationship such as an issue, request, trace, incident, or attachment identifier. Similar wording is not a relationship and is never a deduplication decision.

## Normalized Defect Contract

Use this field shape for every downstream defect:

```yaml
defect_id: <task-local record ID>
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
source_claims: []
missing_fields: []
conflicts: []
uncertainties: []
```

`defect_id` identifies this normalized record only. It is not a canonical ID or deduplication key. Keep plural fields as lists even when only one value exists. Use `unknown` for an absent scalar and an empty list for an absent collection; also name important gaps under `missing_fields`.

## Evidence and Traceability

For every important normalized fact, keep an evidence reference:

```yaml
field: <normalized field>
value: <normalized value>
original_value: <source value>
source_ref:
  source_type: <string>
  source_id: <string|unknown>
  source_uri: <string|unknown>
locator: <line, event, JSON path, attachment, or other stable locator>
```

Keep direct observations in normalized fields. Put an unverified causal or subjective statement under `source_claims`. Put your unresolved interpretation under `uncertainties`.

Do not invent a version, environment, expected behavior, timestamp, source ID, link, or Tool result. When sources disagree, preserve every value and source reference under `conflicts`; do not select a winner for presentation convenience.

This Skill does not form an evaluative verdict, so it does not relabel facts as supporting or contradicting evidence. Downstream evaluative Skills must make that classification against a specific candidate claim.

## Tool Capability Dependencies

Use external capabilities only to retrieve or parse source material. Request capabilities by intent and do not assume a concrete Tool name or API exists.

You may need capabilities to retrieve issue records, tickets, feedback, logs, files, or attachments and to preserve stable raw-content references. If a capability is unavailable, use only supplied content, record the requested capability and error under `tool_failures`, and never invent retrieved data.

Tools provide source facts. They do not normalize away conflicts, compare defects, or infer root causes.

## Procedure

### 1. Validate Sources

Confirm that at least one source contains usable content. Preserve provided identifiers, timestamps, and metadata. Mark an unrecognized type as `unknown_source`.

If no usable content is available, return `processing_status: blocked`, no defects, and a targeted information request. Do not create a fictional defect.

### 2. Interpret Source Semantics

Identify which facts each source can support. For example:

- an issue or ticket can support reported behavior, reproduction, environment, and reporter claims;
- a log can support timestamped signals, error signatures, stack frames, component, and correlation identifiers;
- a test failure can support command, assertion, expected and observed results, revision, and failure output.

Do not treat a reporter diagnosis as verified root cause or a log signal as proof of user impact without a traceable link.

### 3. Extract Facts

Extract source-supported symptoms, behavior, triggers, reproduction, errors, stack traces, affected scope, occurrence, versions, and environments. Attach evidence references to material facts.

### 4. Normalize Conservatively

Normalize whitespace, casing, timestamps, units, component names, and equivalent error formatting only when meaning is preserved. Keep the original value in its evidence reference.

Remove presentation boilerplate and exact repetition, but retain diagnostic context such as identifiers, timestamps, versions, stack frames, assertions, and error codes.

### 5. Record Gaps and Conflicts

List important absent fields. Preserve incompatible values with their sources. Use `unknown` when choosing a value would hide a conflict. Record cleanup decisions under `normalization_notes`.

### 6. Assess Record Completeness

Assign each defect:

- `sufficient`: enough traceable facts for downstream review;
- `partial`: usable for review but materially incomplete or conflicting;
- `insufficient`: a stable defect object exists, but evidence is too sparse for reliable comparison.

Completeness describes input quality, not a duplicate or root-cause verdict. Ask for the exact missing information needed.

### 7. Produce the Result

Return:

```yaml
normalization_result:
  task_id: <string|null>
  processing_status: <completed|blocked>
  defects:
    - <Normalized Defect Contract fields>
      completeness:
        status: <sufficient|partial|insufficient>
        reasons: []
      raw_refs: []
      normalization_notes: []
  unprocessed_sources: []
  information_requests: []
  tool_failures:
    - capability: <requested capability>
      operation_or_query: <what was attempted>
      error: <observed error>
      impact: <effect on normalization>
  next_action: <dispatch_review|request_source|retry_tool>
```

Use `completed` when at least one traceable defect was produced, even if its completeness is `partial` or `insufficient`. Use `blocked` only when a critical prerequisite prevents any honest normalization.
Use `dispatch_review` for usable completed records, `request_source` when source evidence must be supplied, and `retry_tool` when a failed retrieval capability is the primary blocker.

## Failure Handling

| Condition | Required response |
|---|---|
| No usable source content | Return `blocked`, no defect, and request content. |
| Missing source ID or URI | Preserve `unknown`, flag incomplete traceability, and request a stable identifier. |
| Unknown source type | Preserve the provided type in notes, use `unknown_source`, and extract conservatively. |
| Conflicting sources | Preserve all values and references under `conflicts`; never overwrite silently. |
| Truncated evidence | Preserve the available part, mark it incomplete, and request the remainder. |
| Retrieval or parsing capability fails | Record `tool_failures`; continue only if supplied content is sufficient. |
| Content is sparse but traceable | Return `completed` with record completeness `insufficient`; do not invent fields. |

## Example

Input:

```yaml
task_id: normalize-upload-001
raw_sources:
  - source_type: issue
    source_id: ISSUE-2048
    source_uri: https://example.invalid/issues/2048
    timestamp: "2026-08-10T09:00:00Z"
    content: "Uploading files larger than 2GB returns HTTP 500. Request ID req-77."
    metadata:
      version: null
      environment: production
      component: upload-service
  - source_type: log
    source_id: LOG-3321
    source_uri: logs://upload/LOG-3321
    timestamp: "2026-08-10T09:00:03Z"
    content: "req-77 ConnectionResetError component=upload-service"
    metadata:
      version: null
      environment: production
      component: upload-service
```

Output:

```yaml
normalization_result:
  task_id: normalize-upload-001
  processing_status: completed
  defects:
    - defect_id: normalize-upload-001-record-1
      title: Uploads larger than 2 GB return HTTP 500
      summary: A linked issue and log report a large-upload failure.
      symptom: Upload fails above 2 GB.
      expected_behavior: unknown
      actual_behavior: HTTP 500 followed by ConnectionResetError.
      trigger_conditions: [File size larger than 2 GB]
      reproduction_steps: []
      error_signatures: [HTTP 500, ConnectionResetError]
      stack_traces: []
      affected:
        components: [upload-service]
        versions: []
        environments: [production]
        scope: [large-file upload]
      occurrence:
        first_seen: "2026-08-10T09:00:00Z"
        last_seen: "2026-08-10T09:00:03Z"
      source_refs:
        - source_type: issue
          source_id: ISSUE-2048
          source_uri: https://example.invalid/issues/2048
        - source_type: log
          source_id: LOG-3321
          source_uri: logs://upload/LOG-3321
      evidence_refs:
        - field: error_signatures
          value: ConnectionResetError
          original_value: ConnectionResetError
          source_ref: {source_type: log, source_id: LOG-3321, source_uri: logs://upload/LOG-3321}
          locator: content token 2
      source_claims: []
      missing_fields: [expected_behavior, reproduction_steps, affected.versions]
      conflicts: []
      uncertainties:
        - The sources are grouped only because both contain request ID req-77.
      completeness:
        status: partial
        reasons: [Behavior is traceable, but version and reproduction are missing.]
      raw_refs: [ISSUE-2048 content, LOG-3321 content]
      normalization_notes: [Normalized 2GB to 2 GB and preserved the original.]
  unprocessed_sources: []
  information_requests: [Provide the version and reproduction steps.]
  tool_failures: []
  next_action: dispatch_review
```

The output preserves the explicit source link without comparing it with another defect or inferring a root cause.
