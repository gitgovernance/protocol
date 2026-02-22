<!--
  Copyright 2025-2026 GitGovernance (https://www.gitgovernance.com)
  SPDX-License-Identifier: Apache-2.0

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.

  Part of the GitGovernance Protocol — designed and maintained by GitGovernance.
-->

# RFC-06: Execution Record

> Version: 1.0 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-04\
> Schema: `schemas/execution_record_schema.yaml`

## Abstract

The Execution Record is the atomic unit of work in GitGovernance. While a Task defines intent, an Execution records action — a tangible deliverable, a decision, an impediment, or a correction. Every Execution is immutable, cryptographically signed via the Embedded Metadata envelope (RFC-01), and bound to exactly one Task. Together, the sequence of Execution Records for a Task forms a complete, auditable timeline of how work was actually performed.

---

## 1. Motivation

A Task tells you *what* needs to be done. But organizations also need to know *what actually happened* — who did the work, when, what was delivered, what went wrong, and how mistakes were corrected.

The Execution Protocol is more than a simple log. It is the **universal event stream and the single source of truth for the history of work** in GitGovernance. By unifying acts of work (e.g. a commit) and system events (e.g. a state change) into a single chronological log, it enables:

- **Complete traceability.** Reconstruct the full history of a Task — from its first analysis to its final completion — by reading a single stream of records.
- **Real-time monitoring.** A live activity stream can be built by watching new Execution Records as they arrive.
- **Architectural simplicity.** No separate event protocol is needed. Executions *are* the events.
- **Advanced metrics.** Cycle time, time-in-blocker, deployment frequency — all derivable from the execution stream.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Execution Record** | An immutable record of a single act of work performed against a Task. |
| **Execution Type** | A semantic classifier (`analysis`, `progress`, `blocker`, `completion`, `info`, `correction`) that describes the nature of the work. |
| **Typed Reference** | A prefixed string (e.g. `commit:abc123`, `file:src/main.ts`) that links an execution to external evidence. |
| **Correction** | A new Execution Record of type `correction` that supersedes or amends a previous one. The original is never modified. |
| **Metadata** | An optional structured object for machine consumption (audit findings, metrics, scan results) that complements the human-readable `result` and `notes`. |

---

## 3. ID Format

Every Execution Record has a unique identifier following this pattern:

```
{timestamp}-exec-{slug}
```

| Segment | Description |
|:--------|:------------|
| `timestamp` | Unix epoch in seconds (10 digits). Taken from the `author` signature's timestamp. |
| `exec` | Fixed literal. |
| `slug` | A human-readable string derived from the `title` field (see §3.1). |

**Pattern:** `^\d{10}-exec-[a-z0-9-]{1,50}$`
**Max length:** 66 characters

### 3.1. Slug Generation Algorithm

```
FUNCTION generateSlug(title):
  slug = title
    |> lowercase
    |> take_first(50)
    |> replace(/[^a-z0-9]+/g, '-')
    |> trim('-')
  RETURN slug
```

The slug provides context in file listings and log output without requiring a database lookup.

---

## 4. Record Schema

The Execution Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"execution"`. The schema is defined in `execution_record_schema.yaml`.

### 4.1. Required Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `id` | string | `^\d{10}-exec-[a-z0-9-]{1,50}$`, maxLength: 66 | Unique identifier (see §3). |
| `taskId` | string | `^\d{10}-task-[a-z0-9-]{1,50}$`, maxLength: 66 | The Task this execution belongs to. |
| `type` | string | `^(analysis\|progress\|blocker\|completion\|info\|correction\|custom:[a-z0-9-]+)$` | Semantic classification (see §5). |
| `title` | string | 1–256 characters | Human-readable summary. Used to generate the ID slug. |
| `result` | string | minLength: 10 | The tangible deliverable or outcome (see §6). |

### 4.2. Optional Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `notes` | string | — | Narrative, context, and decisions behind the work (see §6). |
| `references` | array of strings | items minLength: 1, maxLength: 500 | Typed links to commits, files, PRs, and external resources (see §7). |
| `metadata` | object | additionalProperties: true | Structured data for programmatic consumption (see §6). |

No additional properties are allowed on the payload object.

---

## 5. Execution Types

The `type` field classifies the nature of the work. The protocol defines six standard types that cover the universal lifecycle of work. Organizations may also define custom types for domain-specific needs.

### Standard Types

| Type | Purpose |
|:-----|:--------|
| `analysis` | Records planning, investigation, or a plan of action before implementation begins. |
| `progress` | Records a tangible, verifiable advancement — code written, a document completed, a sub-task finished. |
| `blocker` | Documents an impediment that prevents further progress on the Task. |
| `completion` | Signals that all deliverables for the Task are finished and acceptance criteria are met. |
| `info` | Records a key decision, finding, or context change that doesn't fit the other types. |
| `correction` | Supersedes or amends a previous Execution Record. The original record is never modified. |

### Custom Types

Organizations may define additional types using the `custom:` prefix (e.g. `custom:deployment`, `custom:rollback`, `custom:security-scan`). Three rules apply:

1. Every conformant implementation must understand the six standard types.
2. Custom types are valid but do not guarantee interoperability between tools.
3. If a tool encounters an unrecognized `custom:*` type, it must treat it as `info` — never reject it.

### 5.1. Type Semantics

**`analysis`** — Used before implementation starts. Typically references documents or design artifacts rather than code commits. The `result` describes the analysis completed; `notes` explains the options evaluated.

**`progress`** — Used during implementation. Should include verifiable evidence (commits, PRs, files). The `result` quantifies the advancement when possible.

**`blocker`** — The `result` must clearly describe what is blocked and why. The `references` should link to the external dependency causing the block (status pages, support tickets). The associated Task should transition to `paused` per the active Workflow.

**`completion`** — Used only when *all* work on the Task is done. Summarizes the final state, includes metrics (coverage, performance), and links to merged PRs or deployments.

**`info`** — For decisions or findings that are important to preserve in the timeline but are not formal analysis, progress, or blockers. Examples: a change in technical strategy, a critical discovery during development.

**`correction`** — Must reference the original Execution Record using the `exec:` typed reference prefix. The `result` states what was wrong and provides the corrected information.

---

## 6. Field Semantics: result, notes, metadata

These three fields serve distinct purposes and should not be conflated.

| Field | Question | Content | Audience |
|:------|:---------|:--------|:---------|
| `result` | **WHAT** was delivered? | The tangible, verifiable outcome. | Humans and tools |
| `notes` | **HOW** and **WHY**? | Narrative context, decisions, alternatives considered. | Humans and future developers |
| `metadata` | **DATA** for machines? | Structured JSON for programmatic consumption. | Tools, agents, pipelines |

### 6.1. The `result` Field

The `result` is the heart of an Execution Record. It must contain verifiable evidence of what was accomplished, not vague descriptions of activity.

- "Refactored 3 N+1 queries into a single optimized JOIN. Response time improved from 2.5s to 200ms." — Good.
- "Worked on the login today." — Not verifiable. Rejected.

### 6.2. The `metadata` Field as Extension Point

The `metadata` field is an open object designed for cases where the output needs to be processed programmatically. It does not replace `result` or `notes` — it complements them.

Common use cases:

- **Audit findings:** `{ "findings": [...], "scannedFiles": 245, "summary": { "critical": 3 } }`
- **Performance metrics:** `{ "duration_ms": 1250, "memory_mb": 512 }`
- **Scan results:** `{ "endpoints_scanned": 15, "vulnerabilities": [...] }`

This makes the Execution Record a natural integration point for vertical products (audit tools, security scanners, CI pipelines) that need to attach structured data to the work timeline.

The protocol intentionally does not define schemas for `metadata` contents. Domain-specific structure — audit findings, agent activity traces, deployment reports — is the responsibility of the consuming product or organization. This keeps the Execution Record generic at the protocol level while enabling unbounded specialization through a single record type.

---

## 7. Typed References

The `references` array uses typed prefixes to create a traceable network linking executions to external evidence.

| Prefix | Purpose | Example |
|:-------|:--------|:--------|
| `commit:` | Git commit SHA | `commit:a1b2c3d4e5f6` |
| `pr:` | Pull Request | `pr:123` |
| `issue:` | GitHub Issue | `issue:456` |
| `file:` | File path (relative to repo root) | `file:src/auth/jwt.ts` |
| `url:` | External resource | `url:https://status.payments.com` |
| `task:` | Related TaskRecord | `task:1752274500-task-auth` |
| `exec:` | Related ExecutionRecord | `exec:1752275500-exec-refactor` |

Unprefixed references are ambiguous and should be avoided. A reference like `"abc123"` could be a commit, an ID, or something else entirely.

---

## 8. Core Rules

### 8.1. Creation

- Every Execution Record must be associated with a valid, existing `taskId`.
- The `id` is generated algorithmically. The `timestamp` portion is taken from the `author` signature in the Embedded Metadata header.
- Every Execution Record must be wrapped in an Embedded Metadata envelope (RFC-01) with `header.type` set to `"execution"`.

### 8.2. Immutability

An Execution Record is a record of a past event and is immutable. Once signed, it cannot be modified. To correct an error, create a new Execution Record of type `correction` that references the original via `exec:{originalId}`.

### 8.3. Validation

Execution Records are validated at two levels:

- **Schema validation:** The record must conform to `execution_record_schema.yaml`.
- **Referential integrity:** The `taskId` must reference an existing Task Record.

These validations are in addition to the Three Gates defined in RFC-01 (integrity, schema, authentication) which apply to the Embedded Metadata envelope.

---

## 9. Persistence

Execution Records are persisted as JSON files wrapped in the Embedded Metadata Container (RFC-01):

```
.gitgov/executions/<executionId>.json
```

Examples:
- `.gitgov/executions/1752274600-exec-implement-oauth.json`
- `.gitgov/executions/1752348000-exec-progress-auth.json`

---

## 10. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to ExecutionRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `TASK_NOT_FOUND` | The `taskId` does not exist | `"Task {taskId} not found in backlog"` |
| `DUPLICATE_ID` | The `executionId` already exists | `"Execution {id} already exists"` |

---

## 11. Examples

### 11.1. Progress — Tangible advancement

A developer completes a performance optimization:

```json
{
  "header": {
    "version": "1.0",
    "type": "execution",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:senior-dev",
        "role": "author",
        "notes": "Refactored N+1 queries in search endpoint",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752275500
      }
    ]
  },
  "payload": {
    "id": "1752275500-exec-refactor-queries",
    "taskId": "1752274500-task-optimizar-api",
    "type": "progress",
    "title": "Refactor de queries N+1",
    "result": "Refactored 3 N+1 queries into a single optimized JOIN. Response time improved from 2.5s to 200ms.",
    "notes": "Identified 3 N+1 queries in the /api/search endpoint. Applied eager loading and relation caching.",
    "references": ["commit:b2c3d4e", "file:src/api/search.ts"]
  }
}
```

### 11.2. Blocker — Impediment documented

An agent records an external dependency failure:

```json
{
  "header": {
    "version": "1.0",
    "type": "execution",
    "payloadChecksum": "b2c3d4e5f6a1789012345678901234567890123456789012345678901234efgh",
    "signatures": [
      {
        "keyId": "agent:camilo:cursor",
        "role": "author",
        "notes": "Payment API down, integration testing blocked",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752361200
      }
    ]
  },
  "payload": {
    "id": "1752361200-exec-payment-api-down",
    "taskId": "1752274500-task-optimizar-api",
    "type": "blocker",
    "title": "Payment API Down",
    "result": "Integration testing blocked. Third-party payment API returning 503 Service Unavailable.",
    "notes": "Contacted provider support (ticket #98765). Incident confirmed on their status page. ETA: 2-3 hours. Continuing with unit tests that don't require the external API.",
    "references": ["url:https://status.payments.com"]
  }
}
```

### 11.3. Analysis with metadata — Automated audit scan

A security agent performs a source code audit and attaches structured findings:

```json
{
  "header": {
    "version": "1.0",
    "type": "execution",
    "payloadChecksum": "c3d4e5f6a1b2789012345678901234567890123456789012345678901234ijkl",
    "signatures": [
      {
        "keyId": "agent:source-audit",
        "role": "author",
        "notes": "Automated GDPR compliance scan completed",
        "signature": "nO0CqR1dUlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXA==",
        "timestamp": 1752276000
      }
    ]
  },
  "payload": {
    "id": "1752276000-exec-source-audit-scan",
    "taskId": "1752274500-task-gdpr-compliance",
    "type": "analysis",
    "title": "GDPR Audit Scan - 2025-07-12",
    "result": "Scanned 245 files. Found 10 findings (3 critical, 4 high, 3 medium). See metadata for structured details.",
    "notes": "Scan executed with RegexDetector + HeuristicDetector. LLM calls: 0 (free tier).",
    "references": ["file:src/config/db.ts", "file:src/auth/keys.ts"],
    "metadata": {
      "scannedFiles": 245,
      "scannedLines": 18420,
      "duration_ms": 1250,
      "findings": [
        { "id": "SEC-001", "severity": "critical", "file": "src/config/db.ts", "line": 5, "type": "api_key" },
        { "id": "SEC-003", "severity": "critical", "file": "src/auth/keys.ts", "line": 2, "type": "private_key" },
        { "id": "PII-003", "severity": "critical", "file": "src/payments/stripe.ts", "line": 8, "type": "credit_card" }
      ],
      "summary": { "critical": 3, "high": 4, "medium": 3, "low": 0 }
    }
  }
}
```

The `metadata` field allows other agents (e.g. a task manager) to process findings programmatically without parsing the human-readable `result`.

### 11.4. Correction — Amending a previous record

A correction to fix incorrectly reported metrics:

```json
{
  "header": {
    "version": "1.0",
    "type": "execution",
    "payloadChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234mnop",
    "signatures": [
      {
        "keyId": "human:senior-dev",
        "role": "author",
        "notes": "Correcting performance metrics from previous execution",
        "signature": "mN9BpQ0zTlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyA==",
        "timestamp": 1752275700
      }
    ]
  },
  "payload": {
    "id": "1752275700-exec-correction-metrics",
    "taskId": "1752274500-task-optimizar-api",
    "type": "correction",
    "title": "Correction: Performance metrics",
    "result": "Correction to execution 1752275500-exec-refactor-queries: actual response time was 200ms, not 50ms as originally reported. Improvement is still significant (92%) but accurate numbers matter for metrics.",
    "notes": "Typo in the original execution record. Real improvement was 2.5s to 200ms, not 2.5s to 50ms.",
    "references": ["exec:1752275500-exec-refactor-queries"]
  }
}
```

---

## 12. Security Considerations

**Referential integrity.** The `taskId` must reference an existing Task Record. This prevents orphaned executions that cannot be attributed to any work stream.

**Correction over mutation.** The protocol provides no mechanism to edit a signed Execution Record. Errors are addressed by creating a new record of type `correction` that references the original. This preserves the complete audit trail, including the mistake and its fix.

**Extension via metadata.** The `metadata` field is an open object. Consumers should validate its contents according to their own schemas before trusting it. The protocol guarantees integrity and authorship of the envelope, but the semantic correctness of `metadata` contents is the responsibility of the producing agent.

---

## 13. References

- Schema: `schemas/execution_record_schema.yaml`
- Embedded Metadata: [RFC-01](./01_embedded.md)
- Task Record: [RFC-04](./04_task.md)
- Workflow: [RFC-08](./08_workflow.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
