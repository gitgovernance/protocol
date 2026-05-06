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

# RFC-07: Feedback Record

> Version: 1.1 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-17\
> Schema: `schemas/feedback_record_schema.yaml`

## Abstract

The Feedback Record is the **nervous system of collaboration** in GitGovernance. It transforms interactions — questions, blockers, suggestions, assignments, approvals — into signed, auditable, and actionable data. While other protocols define the work (TaskRecord), the execution (ExecutionRecord), or the governance rules (WorkflowRecord), Feedback defines the **conversation about the work**.

Unlike traditional comment systems, a FeedbackRecord is a cryptographically signed, **completely immutable** record wrapped in an Embedded Metadata envelope (RFC-01). Status changes are modeled as new FeedbackRecords that reference the original, creating auditable conversation threads.

---

## 1. Motivation

In hybrid human-AI teams, the conversations *around* work are as important as the work itself. Traditional systems handle feedback as mutable annotations — editable comments, reassignable fields, deletable threads — making it impossible to audit *who said what, when, and why*.

GitGovernance solves this with three principles:

- **Feedback is immutable.** A blocking review, a question, an assignment — once signed, it cannot be altered. The historical record is sacred.
- **Status changes are new records.** Resolving a blocker creates a *new* FeedbackRecord that references the original, preserving both the problem and the solution as independent, signed artifacts.
- **Assignment is a conversation, not a field.** Assigning a task creates a signed FeedbackRecord with `type: "assignment"`, providing full traceability of responsibility handovers (see also RFC-04 §6.1).

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Feedback Record** | A signed, immutable record representing a structured interaction about a GitGovernance entity. |
| **Feedback ID** | A unique identifier following the pattern `{timestamp}-feedback-{slug}`. |
| **Entity** | The GitGovernance record that the feedback refers to (identified by `entityType` + `entityId`). |
| **Chaining** | The pattern of creating new FeedbackRecords that reference previous ones to form conversation threads. |
| **Resolution** | A FeedbackRecord that addresses a previous one, using `resolvesFeedbackId` for explicit traceability. |

---

## 3. Feedback ID Structure

### 3.1. ID Format

Every Feedback Record MUST have a globally unique identifier following the pattern:

```
{timestamp}-feedback-{slug}
```

- **`timestamp`**: Unix epoch seconds (integer, 10 digits) at the time of record creation. Derived from the author's signature timestamp.
- **`slug`**: A URL-friendly string derived from the content, matching `[a-z0-9-]{1,50}`.

**Pattern**: `^\d{10}-feedback-[a-z0-9-]{1,50}$`
**Max Length**: 70 characters (10 timestamp + 1 dash + 8 "feedback" + 1 dash + 50 slug)

### 3.2. ID Generation Algorithm

```
FUNCTION generateFeedbackId(content, timestamp = now()):
  slug = content
    |> lowercase
    |> take_first(50)
    |> replace(/[^a-z0-9]+/g, '-')
    |> trim('-')
  RETURN "{timestamp}-feedback-{slug}"

EXAMPLES:
  "Blocking: REST API endpoints" → "1752788100-feedback-blocking-rest-api-endpoints"
  "Question about test coverage" → "1752788400-feedback-question-about-test-coverage"
```

*Rationale: Timestamps provide inherent chronological sorting and collision resistance without requiring centralized counters.*

---

## 4. Record Schema

The Feedback Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"feedback"`. The schema is defined in `feedback_record_schema.yaml`.

### 4.1. Mandatory Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `id` | string | `^\d{10}-feedback-[a-z0-9-]{1,50}$`, maxLength: 70 | Unique identifier (§3). |
| `entityType` | enum | `actor` \| `agent` \| `task` \| `execution` \| `feedback` \| `cycle` \| `workflow` | The type of entity this feedback refers to. |
| `entityId` | string | minLength: 1, maxLength: 256 | The ID of the referenced entity. Must match the ID pattern for its `entityType` (§4.3). |
| `type` | enum | `blocking` \| `suggestion` \| `question` \| `approval` \| `clarification` \| `assignment` | Semantic intent (§5). |
| `status` | enum | `open` \| `acknowledged` \| `resolved` \| `wontfix` | Lifecycle status (§6). |
| `content` | string | minLength: 1 | The content of the feedback. |

### 4.2. Optional Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `assignee` | string | `^(human\|agent)(:[a-z0-9-]+)+$`, maxLength: 256 | Actor ID responsible for addressing the feedback. |
| `resolvesFeedbackId` | string | `^\d{10}-feedback-[a-z0-9-]{1,50}$`, maxLength: 70 | ID of the FeedbackRecord that this one resolves or responds to. |
| `metadata` | object | additionalProperties: true | Structured data for machine consumption (e.g., waiver details, audit context). |

No additional properties are allowed at the root level.

### 4.3. Entity ID Patterns

The `entityId` value MUST match the ID pattern defined by the corresponding record type:

| `entityType` | `entityId` Pattern |
|:-------------|:-------------------|
| `actor` | `^(human\|agent)(:[a-z0-9-]+)+$` |
| `agent` | `^agent(:[a-z0-9-]+)+$` |
| `task` | `^\d{10}-task-[a-z0-9-]{1,50}$` |
| `execution` | `^\d{10}-exec-[a-z0-9-]{1,50}$` |
| `feedback` | `^\d{10}-feedback-[a-z0-9-]{1,50}$` |
| `cycle` | `^\d{10}-cycle-[a-z0-9-]{1,50}$` |
| `workflow` | `^\d{10}-workflow-[a-z0-9-]{1,50}$` |

*Note: Actor and Agent IDs follow a hierarchical format (`type:segments`) rather than the timestamp-based pattern used by other record types. See RFC-02 §3 and RFC-03 §3.*

---

## 5. Feedback Types

The `type` field classifies the semantic intent of the feedback:

| Type | Purpose | Implication |
|:-----|:--------|:------------|
| `blocking` | Impedes progress. Requires resolution to continue. | **Immediate action required.** |
| `suggestion` | Proposes an improvement or non-blocking alternative. | Recommendation to consider. |
| `question` | Requests clarification or additional information. | Needs a response to unblock. |
| `approval` | Expresses agreement or validates a deliverable/proposal. | Positive signal; may unblock. |
| `clarification` | Responds to a previous `question` or adds context. | Provides information. |
| `assignment` | Assigns an entity (typically a Task) to an actor. | Defines responsibility. |

### 5.1. Blocking (Feedback) vs Blocker (Execution)

These two types share similar names but represent **opposite directions of intent**:

- **FeedbackRecord `type: "blocking"`** — An *external intervention*. A reviewer, auditor, or lead tells the executor: "I am blocking you because of X." This is a governance gate — a Workflow (RFC-08) can require its resolution before allowing a state transition.

- **ExecutionRecord `type: "blocker"`** (RFC-06) — A *self-declaration*. The executor working on a task documents: "I am blocked by X." This is a work log entry — it records an obstacle encountered during execution but does not impose a gate on others.

Both are valid and complementary. A task can simultaneously have a `blocker` ExecutionRecord (the developer reporting a dependency issue) and a `blocking` FeedbackRecord (a reviewer flagging a quality concern).

---

## 6. Feedback Status

The `status` field tracks the lifecycle of the feedback:

| Status | Meaning |
|:-------|:--------|
| `open` | The feedback has been created and needs attention. |
| `acknowledged` | The feedback has been seen and is planned to be addressed. |
| `resolved` | The feedback has been satisfactorily addressed. |
| `wontfix` | A decision has been made not to act on the feedback. |

### 6.1. Initial Status Rules

- Conversational types (`blocking`, `question`, `suggestion`) are created with `status: "open"`.
- Feedbacks of type `approval` are typically created with `status: "resolved"`.
- Feedbacks of type `assignment` may be created with any status depending on context.
- Feedbacks that respond to others (using `resolvesFeedbackId`) are typically created with `status: "resolved"`.

---

## 7. Governance Protocols

### 7.1. Complete Immutability

A FeedbackRecord is **completely immutable**. Once created, no field may be modified — including `status`.

**To change the status of a feedback:**

1. Create a **new FeedbackRecord** that documents the status change.
2. Set `entityType: "feedback"` and `entityId` to the ID of the original feedback.
3. Use `type: "clarification"` or `"approval"` as appropriate.
4. Use `resolvesFeedbackId` for explicit traceability.
5. In `content`, document the resolution.

*Rationale: Immutability guarantees that the historical record cannot be rewritten. Every status change is a new signed event, creating an auditable chain of decisions.*

### 7.2. Conversation Chaining

Responses and resolutions are managed through **chaining**:

- A new FeedbackRecord points to the original via `entityType: "feedback"` + `entityId`.
- Optionally uses `resolvesFeedbackId` for explicit resolution traceability.
- This creates immutable, auditable conversation threads.

Multiple feedbacks can reference the same original, creating branching discussions. Each branch is independently signed and traceable.

### 7.3. Entity Validation

A FeedbackRecord MUST be associated with a valid `entityType` and `entityId`. Implementations SHOULD validate that the referenced entity exists before creating the feedback.

### 7.4. Assignment Semantics

A feedback with `type: "assignment"` is the canonical mechanism for assigning responsibility in GitGovernance. The `assignee` field identifies the actor being assigned.

**Why assignment is a FeedbackRecord, not a field on TaskRecord:**

- **Full traceability.** Each assignment and reassignment is a signed, timestamped event — creating an immutable ledger of responsibility.
- **Decoupling.** The TaskRecord stays focused on the promise of work, while the FeedbackRecord manages execution logistics.
- **Explicit governance.** Assignment becomes a formal ceremony — an auditable act of communication rather than a metadata edit.

See also RFC-04 §6.1 for how Tasks handle assignment.

---

## 8. Persistence

Feedback Records are persisted as JSON files wrapped in the Embedded Metadata envelope (RFC-01):

```
.gitgov/feedbacks/<feedbackId>.json
```

Examples:
- `.gitgov/feedbacks/1752788100-feedback-blocking-rest-api.json`
- `.gitgov/feedbacks/1752788300-feedback-assign-auth-task.json`

---

## 9. Verification

Integrity of a Feedback Record is verified through three checks:

1. **Payload Checksum**: The SHA-256 checksum of the canonically serialized payload matches `header.payloadChecksum`.
2. **Schema Validation**: The payload conforms to `feedback_record_schema.yaml`.
3. **Entity Reference**: The `entityId` references a valid, existing record of the declared `entityType`.

---

## 10. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to FeedbackRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `ENTITY_NOT_FOUND` | Referenced `entityId` does not exist | `"Entity {entityId} not found"` |
| `DUPLICATE_ID` | Feedback ID already exists | `"Feedback {id} already exists"` |

---

## 11. Examples

### 11.1. Blocking Feedback on Execution

A reviewer identifies a critical issue in an execution:

```json
{
  "header": {
    "version": "1.0",
    "type": "feedback",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:jorge",
        "role": "author",
        "notes": "Blocking feedback: REST endpoint implementation does not follow standards",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752788100
      }
    ]
  },
  "payload": {
    "id": "1752788100-feedback-blocking-rest-api",
    "entityType": "execution",
    "entityId": "1752642000-exec-subtarea-9-4",
    "type": "blocking",
    "status": "open",
    "content": "This implementation does not comply with the REST endpoint standard. Endpoints must follow the /api/v1/{resource}/{id} pattern. Currently uses /get-user?id=X which is not RESTful."
  }
}
```

### 11.2. Feedback Resolution (Chaining)

An agent resolves the blocking feedback by creating a new record:

```json
{
  "header": {
    "version": "1.0",
    "type": "feedback",
    "payloadChecksum": "b2c3d4e5f6a1789012345678901234567890123456789012345678901234efgh",
    "signatures": [
      {
        "keyId": "agent:camilo:cursor",
        "role": "author",
        "notes": "Resolved blocking feedback: REST endpoints refactored to standard pattern",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752788200
      }
    ]
  },
  "payload": {
    "id": "1752788200-feedback-rest-api-fixed",
    "entityType": "feedback",
    "entityId": "1752788100-feedback-blocking-rest-api",
    "type": "clarification",
    "status": "resolved",
    "content": "Fix implemented. All endpoints now follow REST standard: GET /api/v1/users/:id, POST /api/v1/users, etc. Tests updated and passing.",
    "resolvesFeedbackId": "1752788100-feedback-blocking-rest-api"
  }
}
```

### 11.3. Task Assignment

A tech lead assigns a task to a developer:

```json
{
  "header": {
    "version": "1.0",
    "type": "feedback",
    "payloadChecksum": "c3d4e5f6a1b2789012345678901234567890123456789012345678901234ijkl",
    "signatures": [
      {
        "keyId": "human:tech-lead",
        "role": "author",
        "notes": "Assigning OAuth task to Maria based on domain expertise",
        "signature": "mN9BpQ0zTlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyA==",
        "timestamp": 1752788300
      }
    ]
  },
  "payload": {
    "id": "1752788300-feedback-assign-auth-task",
    "entityType": "task",
    "entityId": "1752274500-task-implementar-oauth",
    "type": "assignment",
    "status": "open",
    "content": "Assigning this task to Maria for her experience with OAuth2. High priority for the current sprint.",
    "assignee": "human:maria"
  }
}
```

### 11.4. Question on Task

An agent asks for clarification on task requirements:

```json
{
  "header": {
    "version": "1.0",
    "type": "feedback",
    "payloadChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234mnop",
    "signatures": [
      {
        "keyId": "agent:code-reviewer",
        "role": "author",
        "notes": "Requesting clarification on test coverage expectations",
        "signature": "nO0CqR1dUlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXA==",
        "timestamp": 1752788400
      }
    ]
  },
  "payload": {
    "id": "1752788400-feedback-question-test-coverage",
    "entityType": "task",
    "entityId": "1752274500-task-implementar-oauth",
    "type": "question",
    "status": "open",
    "content": "What level of test coverage is expected for this feature? The spec does not mention it explicitly. Should we aim for 80% like the rest of the project?"
  }
}
```

### 11.5. Waiver with Metadata (Audit)

A security lead approves a waiver with structured audit metadata:

```json
{
  "header": {
    "version": "1.0",
    "type": "feedback",
    "payloadChecksum": "e5f6a1b2c3d4789012345678901234567890123456789012345678901234qrst",
    "signatures": [
      {
        "keyId": "human:security-lead",
        "role": "author",
        "notes": "Waiver approved: PII-001 finding is a test fixture, not real PII",
        "signature": "pQ2DrS3eVlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXA==",
        "timestamp": 1752788500
      }
    ]
  },
  "payload": {
    "id": "1752788500-feedback-waiver-pii-001",
    "entityType": "execution",
    "entityId": "1752276000-exec-source-audit-scan",
    "type": "approval",
    "status": "resolved",
    "content": "Waiver approved: This email is a test placeholder, not real PII.",
    "metadata": {
      "fingerprint": "abc123def456",
      "ruleId": "PII-001",
      "file": "src/test/fixtures/user.ts",
      "line": 42,
      "expiresAt": "2025-12-31T23:59:59Z"
    }
  }
}
```

---

## 12. Security Considerations

**Immutability.** FeedbackRecords are fully immutable. Once signed, no field — including `status` — can be altered without invalidating the cryptographic envelope. Status changes are modeled as new records, preserving the complete history.

**Conversation integrity.** The chaining model ensures that conversation threads cannot be forged or reordered. Each link in the chain is independently signed and timestamped.

**Assignment auditability.** Because assignment is a signed FeedbackRecord rather than a mutable field, every handover of responsibility is cryptographically attributable. This prevents disputes about who was assigned what, and when.

**Entity binding.** A FeedbackRecord is permanently bound to its target entity via `entityType` and `entityId`. This binding is protected by the payload checksum and cannot be silently redirected.

---

## 13. References

- Schema: `schemas/feedback_record_schema.yaml`
- Embedded Metadata: [RFC-01](./01_embedded.md)
- Actor (identity for signatures): [RFC-02](./02_actor.md)
- Task (assignment model): [RFC-04](./04_task.md)
- Execution (work records): [RFC-06](./06_execution.md)
- Workflow (transition rules): [RFC-08](./08_workflow.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
