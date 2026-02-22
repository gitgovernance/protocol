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

# RFC-05: Cycle Record

> Version: 1.0 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-20\
> Schema: `schemas/cycle_record_schema.yaml`

## Abstract

The Cycle Record is the **strategic grouping primitive** within the GitGovernance Protocol. It defines *why* work is being done by grouping TaskRecords into coherent units — such as sprints, milestones, or quarterly roadmaps. Unlike the Task (RFC-04) which models the atomic unit of work, a CycleRecord models the *initiative* that gives tasks their shared purpose. Cycles are cryptographically signed, immutable records wrapped in an Embedded Metadata envelope (RFC-01). They support hierarchical composition via `childCycleIds` and bidirectional linkage with tasks via the `taskIds`/`cycleIds` pair.

---

## 1. Motivation

In hybrid human-AI teams, strategic alignment is as critical as task execution. Without a formal grouping mechanism:

- **Work becomes disconnected.** Individual tasks lose their shared context and purpose.
- **Progress is unmeasurable.** Tracking completion of individual tasks gives no visibility into strategic milestones.
- **AI agents lack boundaries.** Autonomous agents need a defined "playing field" with clear objectives and scope to make tactical decisions.

GitGovernance solves this with three principles:

- **Cycles define purpose.** Tasks define *what* is done; Cycles define *why* it is done. A cycle groups work under a shared strategic objective.
- **Grouping is bidirectional.** A Cycle references its tasks via `taskIds`, and tasks reference their cycles via `cycleIds` (RFC-04). This cross-referencing enables navigation in both directions.
- **Cycles compose hierarchically.** A quarterly roadmap can contain sprints, which contain individual tasks. The `childCycleIds` field enables multi-level planning without complex nesting rules.
- **Cycles are methodology-agnostic.** They can represent sprints, milestones, bets, phases, or any organizational grouping. The protocol defines the container; the team defines its semantics.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Cycle Record** | A signed, immutable record representing a strategic grouping of work. |
| **Cycle ID** | A unique identifier following the pattern `{timestamp}-cycle-{slug}`. |
| **Lifecycle State** | One of the 4 states in the cycle lifecycle (`planning`, `active`, `completed`, `archived`). |
| **Child Cycle** | A cycle that is part of a hierarchical grouping via `childCycleIds`. |
| **Container Cycle** | A cycle with `childCycleIds` but no `taskIds` — used for roadmaps and epics. |
| **Bidirectional Link** | The cross-reference between `CycleRecord.taskIds` and `TaskRecord.cycleIds`. |

---

## 3. Cycle ID Structure

### 3.1. ID Format

Every Cycle MUST have a globally unique identifier following the pattern:

```
{timestamp}-cycle-{slug}
```

- **`timestamp`**: Unix epoch seconds (integer, 10 digits) at the time of record creation. Derived from the author's signature timestamp.
- **`slug`**: A URL-friendly string derived from the title, matching `[a-z0-9-]{1,50}`.

**Pattern**: `^\d{10}-cycle-[a-z0-9-]{1,50}$`
**Max Length**: 67 characters (10 timestamp + 1 dash + 5 "cycle" + 1 dash + 50 slug)

### 3.2. ID Generation Algorithm

```
FUNCTION generateCycleId(title, timestamp = now()):
  slug = title
    |> lowercase
    |> take_first(50)
    |> replace(/[^a-z0-9]+/g, '-')
    |> trim('-')
  RETURN "{timestamp}-cycle-{slug}"

EXAMPLES:
  "Sprint 24 - API Performance" → "1754400000-cycle-sprint-24-api-performance"
  "Q4 2025 - Growth & Scale"    → "1754600000-cycle-q4-2025-growth-scale"
```

*Rationale: Same algorithm as Task IDs (RFC-04 §3.2) with the `cycle` infix, ensuring consistent ID generation across all record types.*

---

## 4. Record Schema

The Cycle Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"cycle"`. The schema is defined in `cycle_record_schema.yaml`.

### 4.1. Mandatory Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `id` | string | `^\d{10}-cycle-[a-z0-9-]{1,50}$`, maxLength: 67 | Unique identifier (§3). |
| `title` | string | minLength: 1, maxLength: 256 | Human-readable name for the cycle. |
| `status` | enum | `planning` \| `active` \| `completed` \| `archived` | Current lifecycle state (§5). |

### 4.2. Optional Fields

| Field | Type | Constraint | Default | Description |
|:------|:-----|:-----------|:--------|:------------|
| `taskIds` | array of strings | items: `^\d{10}-task-[a-z0-9-]{1,50}$`, maxLength: 66 | `[]` | Task IDs that belong to this cycle. Bidirectional with `TaskRecord.cycleIds`. |
| `childCycleIds` | array of strings | items: `^\d{10}-cycle-[a-z0-9-]{1,50}$`, maxLength: 67 | `[]` | IDs of child cycles for hierarchical composition (§6.2). |
| `tags` | array of strings | items: `^[a-z0-9-]+(:[a-z0-9-]+)*$`, maxLength per item: 100 | `[]` | `key:value` tags for categorization (e.g. `sprint:24`, `team:backend`). |
| `notes` | string | minLength: 1 | — | Description of the cycle's goals, objectives, and context. |

No additional properties are allowed at the root level.

---

## 5. The State Machine

### 5.1. The 4 Lifecycle States

The Cycle defines **4 lifecycle states**. The typical progression is `planning → active → completed → archived`, but the actual valid transitions are governed by the active Workflow (RFC-08). A Workflow MAY permit non-sequential transitions (e.g. `planning → archived` for a cancelled initiative).

### 5.2. State Descriptions

| State | Meaning | Transition Event |
|:------|:--------|:-----------------|
| `planning` | Scope definition. Tasks and child cycles are being added or removed. | Creation by an author. |
| `active` | Work in progress. Ideally, scope is frozen. | Start — signed by a cycle owner or authorized actor. |
| `completed` | All work finished. Tasks are done or moved to another cycle. | Complete — signed by an authorized actor. |
| `archived` | Historical record; lifecycle closed. | Explicit archive command with appropriate signature. |

### 5.3. Status Independence

Cycle status and Task status are **independent but correlated**. The protocol does NOT enforce that all tasks be `done` before a cycle can transition to `completed`. The decision of "when to complete" is organizational policy, defined in the Workflow (RFC-08):

- Tasks may be moved to a different cycle before completion.
- Tasks may be `discarded` if no longer relevant.
- A Workflow MAY require all tasks to be `done` or `archived` as a gate condition.

---

## 6. Governance Protocols

### 6.1. Bidirectional Linkage with Tasks

A task's `cycleIds` field (RFC-04) and a cycle's `taskIds` field form a **bidirectional relationship**. A task MAY belong to multiple cycles simultaneously — for example, a task can belong to both a weekly sprint and a quarterly milestone.

Implementations SHOULD maintain consistency between both sides of the link. However, referential integrity validation (checking that all referenced IDs exist) is a tool responsibility (e.g. `gitgov audit`), not a protocol requirement.

### 6.2. Hierarchical Composition

Cycles support multi-level hierarchies via `childCycleIds`:

```
Q4 2025 - Growth & Scale (Quarterly)
  ├─ Sprint 24 - API Performance (2 weeks)
  ├─ Sprint 25 - Mobile App (2 weeks)
  └─ Auth System v2.0 (Milestone)
```

**Rules:**
- A cycle MAY have both `taskIds` and `childCycleIds`, or only one of them.
- A cycle with no `taskIds` and only `childCycleIds` is a **container cycle** — valid and common for roadmaps and epics.
- The protocol does NOT enforce acyclicity of the `childCycleIds` graph. Implementations SHOULD detect circular references during audit.

### 6.3. Time-Agnostic Design

The protocol does NOT include explicit date fields (start date, end date, deadline). Temporal information is derived from:

- **Creation time**: The `timestamp` in the first signature of the CycleRecord.
- **Completion time**: The `timestamp` of the signature that transitions the status to `completed`.
- **Duration**: Computed from the difference between creation and completion timestamps.

*Rationale: Embedding dates in the record creates a divergence between "planned dates" and "actual dates" — leading to stale metadata. Deriving timestamps from signatures ensures they reflect reality.*

---

## 7. Persistence

Cycle Records are persisted as JSON files wrapped in the Embedded Metadata Container (RFC-01):

```
.gitgov/cycles/<cycleId>.json
```

Examples:
- `.gitgov/cycles/1754400000-cycle-sprint-24-api-performance.json`
- `.gitgov/cycles/1754600000-cycle-q4-2025-growth.json`

---

## 8. Verification

Integrity of a Cycle Record is verified through three checks:

1. **Payload Checksum**: The SHA-256 checksum of the canonically serialized payload matches `header.payloadChecksum`.
2. **Schema Validation**: The payload conforms to `cycle_record_schema.yaml`.
3. **Referential Integrity** (optional, tool-level): All IDs in `taskIds` exist as TaskRecords. All IDs in `childCycleIds` exist as CycleRecords.

---

## 9. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to CycleRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `TASK_NOT_FOUND` | A taskId does not exist | `"Task {taskId} not found in backlog"` |
| `CYCLE_NOT_FOUND` | A childCycleId does not exist | `"Cycle {cycleId} not found"` |
| `DUPLICATE_ID` | Cycle ID already exists | `"Cycle {id} already exists"` |

---

## 10. Examples

### 10.1. Sprint (2-week iteration, active)

A focused sprint with three performance-related tasks.

```json
{
  "header": {
    "version": "1.0",
    "type": "cycle",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:product-manager",
        "role": "author",
        "notes": "Sprint 24 kicked off — focus on API performance for Black Friday readiness",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1754400000
      }
    ]
  },
  "payload": {
    "id": "1754400000-cycle-sprint-24-api-performance",
    "title": "Sprint 24 - API Performance",
    "status": "active",
    "taskIds": [
      "1752274500-task-optimizar-endpoint-search",
      "1752360900-task-anadir-cache-a-redis",
      "1752447300-task-implementar-rate-limiting"
    ],
    "tags": ["sprint:24", "team:backend", "focus:performance"],
    "notes": "Objective: Reduce API p95 latency below 200ms and prepare infrastructure for Black Friday."
  }
}
```

### 10.2. Milestone (product feature, planning)

A milestone grouping multiple tasks into a cohesive deliverable.

```json
{
  "header": {
    "version": "1.0",
    "type": "cycle",
    "payloadChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234efab",
    "signatures": [
      {
        "keyId": "human:cto",
        "role": "author",
        "notes": "Authentication v2 milestone — critical for Q4 launch",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1754500000
      }
    ]
  },
  "payload": {
    "id": "1754500000-cycle-auth-system-v2",
    "title": "Authentication System v2.0",
    "status": "planning",
    "taskIds": [
      "1752274500-task-oauth2-integration",
      "1752360900-task-2fa-implementation",
      "1752447300-task-password-recovery",
      "1752533700-task-session-management"
    ],
    "tags": ["milestone:v2", "security", "feature:auth"],
    "notes": "Major milestone: Complete authentication system with OAuth2, 2FA, and advanced session management. Critical for Q4 launch."
  }
}
```

### 10.3. Quarterly Roadmap (hierarchical container, active)

A high-level cycle that composes child cycles instead of directly referencing tasks.

```json
{
  "header": {
    "version": "1.0",
    "type": "cycle",
    "payloadChecksum": "a7b8c9a1b2c3789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:ceo",
        "role": "author",
        "notes": "Q4 2025 strategic objectives approved by leadership",
        "signature": "PqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPA==",
        "timestamp": 1754600000
      }
    ]
  },
  "payload": {
    "id": "1754600000-cycle-q4-2025-growth",
    "title": "Q4 2025 - Growth & Scale",
    "status": "active",
    "childCycleIds": [
      "1754400000-cycle-sprint-24-api-performance",
      "1754500000-cycle-auth-system-v2",
      "1754650000-cycle-mobile-app-launch"
    ],
    "tags": ["roadmap:q4", "strategy:growth", "okr:scale-to-1m-users"],
    "notes": "Quarterly objective: Scale to 1M active users. Includes performance improvements, new auth system, and mobile app launch."
  }
}
```

---

## 11. Security Considerations

**Hierarchical integrity.** The `childCycleIds` field creates parent-child relationships between cycles. Implementations MUST prevent circular references — a cycle cannot be its own ancestor. Circular hierarchies would break traversal and reporting.

**Bidirectional consistency.** Cycle-to-task links (`taskIds`) and task-to-cycle links (`cycleIds`) form a bidirectional relationship. Implementations SHOULD verify consistency: if a task appears in a cycle's `taskIds`, that cycle's ID should appear in the task's `cycleIds`. Inconsistencies indicate data corruption or unauthorized modification.

**Status transitions.** Cycle status changes (planning → active → completed → archived) are governed by the Workflow Engine and require valid signatures. Archiving a cycle does not automatically archive its tasks — each task has its own independent lifecycle.

---

## 12. References

- Schema: `schemas/cycle_record_schema.yaml`
- Embedded Metadata: [RFC-01](./01_embedded.md)
- Actor (identity for signatures): [RFC-02](./02_actor.md)
- Task (atomic work unit, bidirectional link): [RFC-04](./04_task.md)
- Workflow (transition rules): [RFC-08](./08_workflow.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
