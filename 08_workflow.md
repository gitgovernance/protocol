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

# RFC-08: Workflow Record

> Version: 1.0 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-17\
> Schema: `schemas/workflow_record_schema.yaml`

## Abstract

The Workflow Record is the declarative "constitution" of a GitGovernance repository. It separates business rules from data storage: while a Task Record stores the current state (e.g. `active`), the Workflow Record defines **named transitions** — each specifying source states, a target state, and the gates required to cross. This architecture enables infinite governance variations — from startup speed to enterprise compliance — without changing the core protocol.

---

## 1. Motivation

Every team has the same fundamental question: *"Can this task move from here to there?"* But the answer depends on who you are, what rules your organization enforces, and what evidence has been produced.

Hardcoding these rules into software creates rigid, one-size-fits-all systems. GitGovernance takes a different approach: the rules are data. A Workflow Record is a signed, immutable configuration that declares which transitions are valid, what signatures are required, and what custom validations must pass. Changing the rules means creating a new signed Workflow Record — the change itself becomes an auditable event.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Workflow Record** | A signed, immutable configuration that defines the governance rules for a repository. |
| **State Transition** | A declared, named movement from one or more source states to a target state, subject to gate requirements. Each transition is identified by a unique name (e.g. `submit`, `approve`, `resume`). |
| **Gate** | A requirement that must be satisfied before a transition is authorized. Three types: Command, Event, Signature. |
| **Double Key Doctrine** | The security mechanism requiring both intent (signature role) and capability (actor roles) to authorize a transition. |
| **Signature Group** | A named set of signature requirements within a transition (e.g. `__default__`, `quality`, `design`). |
| **Custom Rule** | A named validation that extends the protocol with organization-specific logic. |

---

## 3. Record Schema

The Workflow Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"workflow"`. The schema is defined in `workflow_record_schema.yaml`.

### 3.1. Root Fields

| Field | Type | Required | Constraint | Description |
|:------|:-----|:---------|:-----------|:------------|
| `id` | string | Yes | `^\d{10}-workflow-[a-z0-9-]{1,50}$`, maxLength: 70 | Unique identifier following the `{timestamp}-workflow-{slug}` convention. |
| `name` | string | Yes | 1–100 characters | Human-readable name (e.g. "GitGovernance Default", "Scrum Strict"). |
| `description` | string | No | max 500 characters | Brief description of the methodology's purpose. |
| `state_transitions` | object | Yes | — | Map of named transitions to their rules (see §3.2). |
| `custom_rules` | object | No | — | Definitions for custom validation rules (see §6). |
| `agent_integration` | object | No | — | Agent automation configuration (see §9). |

No additional properties are allowed at the root level.

### 3.2. Named Transitions

The `state_transitions` object maps **named transitions** to their rules. Keys are transition names matching `^[a-z][a-z0-9_]{0,49}$` — they identify the transition itself, not the target state. Each transition requires:

| Field | Type | Required | Constraint | Description |
|:------|:-----|:---------|:-----------|:------------|
| `from` | array of strings | Yes | minItems: 1, item pattern: `^[a-z][a-z0-9_]{0,49}$` | Valid source states. |
| `to` | string | Yes | pattern: `^[a-z][a-z0-9_]{0,49}$` | Target state. |
| `requires` | object | Yes | — | The gate requirements for this transition (see §3.3). |

No additional properties are allowed on transition objects.

The schema models transitions as **edges** (named movements between states), not as **nodes** (target states). This allows multiple transitions to share a target state with different requirements — for example, `activate` (from `ready`) and `resume` (from `paused`) both reach `active` but through different gates.

### 3.3. The `requires` Object

Each transition declares what is needed to cross the gate. All fields are optional, but at least one should be present for the transition to be meaningful.

| Field | Type | Required | Constraint | Description |
|:------|:-----|:---------|:-----------|:------------|
| `command` | string | No | — | The CLI command that triggers this transition (Command Gate). |
| `event` | string | No | — | The system event that triggers this transition (Event Gate). |
| `signatures` | object | No | — | Signature group requirements (Signature Gate). See §3.4. |
| `custom_rules` | array of strings | No | — | Identifiers of custom rules that must pass (see §6). |

No additional properties are allowed on the `requires` object.

A transition may combine multiple gate types. For example, a transition might require both a command and signatures — the command initiates the transition, and the signatures authorize it.

### 3.4. Signature Groups

The `signatures` object within `requires` maps group names to their requirements. Each group is independently evaluated. A transition is authorized only when **all** groups have their requirements satisfied.

| Field | Type | Required | Constraint | Description |
|:------|:-----|:---------|:-----------|:------------|
| `role` | string | Yes | — | The signature role required (e.g. `approver`, `submitter`). |
| `capability_roles` | array of strings | Yes | minItems: 1 | Actor capability roles that qualify. |
| `min_approvals` | integer | Yes | minimum: 1 | Minimum number of valid signatures needed. |
| `actor_type` | enum | No | `"human"` \| `"agent"` | Restrict to specific actor type. |
| `specific_actors` | array of strings | No | — | Restrict to specific actor IDs. |

No additional properties are allowed on signature group objects.

The special group name `__default__` is conventional for the primary signature requirement. Additional named groups (e.g. `design`, `quality`, `ai`) represent independent approval tracks.

---

## 4. The Three Gate Types

Every transition in a Workflow is guarded by one or more gates. The protocol defines three types.

### 4.1. Command Gates

Explicit ceremonies that require human or agent intent.

```json
"requires": {
  "command": "gitgov task submit"
}
```

The actor must deliberately invoke the command. The transition does not happen automatically.

### 4.2. Event Gates

Automatic transitions triggered by system events.

```json
"requires": {
  "event": "first_execution_record_created"
}
```

When the event occurs, the system evaluates the transition. No explicit action is needed from the actor.

### 4.3. Signature Gates

Authorization checks that validate both intent and capability (see §5).

```json
"requires": {
  "signatures": {
    "__default__": {
      "role": "approver",
      "capability_roles": ["approver:quality"],
      "min_approvals": 1
    }
  }
}
```

---

## 5. The Double Key Doctrine

The security mechanism that ensures only authorized actors can approve transitions. Every signature is validated against two dimensions:

- **First Key (Intent):** The `role` in the signature itself — "I am signing as an `approver`."
- **Second Key (Capability):** The `roles` array in the signer's Actor Record — "I possess the `approver:quality` capability."

Both keys must match for the signature to count toward the transition's `min_approvals`. This prevents accidental signatures (wrong intent) and unauthorized approvals (insufficient capability).

A signature group may additionally restrict by:
- **`actor_type`**: Only `"human"` or only `"agent"` signatures accepted.
- **`specific_actors`**: Only listed actor IDs accepted.

### 5.1. Multiple Signature Groups

A single transition can require multiple independent approval tracks:

```json
"signatures": {
  "__default__": {
    "role": "approver",
    "capability_roles": ["approver:product"],
    "min_approvals": 1
  },
  "quality": {
    "role": "approver",
    "capability_roles": ["approver:quality"],
    "min_approvals": 1
  },
  "ai": {
    "role": "approver",
    "capability_roles": ["approver:ai"],
    "min_approvals": 2,
    "actor_type": "agent"
  }
}
```

The transition is authorized only when all groups are satisfied simultaneously.

---

## 6. Custom Rules

Organizations can extend the protocol with named validation rules. Each rule is defined in the root `custom_rules` object and referenced by name in transition `requires.custom_rules`.

### 6.1. Rule Definition

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `description` | string | Yes | Human-readable description. Max 200 characters. |
| `validation` | enum | Yes | `assignment_required` &#124; `sprint_capacity` &#124; `epic_complexity` &#124; `custom` |
| `parameters` | object | No | Parameters for the validation. |
| `expression` | string | No | Inline expression for `custom` validations. Must return boolean. |
| `module_path` | string | No | Path to external module (alternative to `expression`). |

No additional properties are allowed on rule definition objects.

### 6.2. Built-in Validation Types

| Type | Purpose |
|:-----|:--------|
| `assignment_required` | The task must have a valid assignment before the transition. |
| `sprint_capacity` | The task must belong to an active sprint/cycle. |
| `epic_complexity` | Complex epics must be decomposed before proceeding. |
| `custom` | Organization-specific logic via `expression` or `module_path`. |

### 6.3. Usage in Transitions

```json
"state_transitions": {
  "activate": {
    "from": ["ready"],
    "to": "active",
    "requires": {
      "event": "first_execution_record_created",
      "custom_rules": ["task_must_have_valid_assignment_for_executor"]
    }
  }
},
"custom_rules": {
  "task_must_have_valid_assignment_for_executor": {
    "description": "Task must have a valid assignment before execution can begin",
    "validation": "assignment_required"
  }
}
```

---

## 7. Entity Lifecycles

The Workflow Record governs state transitions for **any entity type** — not just tasks. The same mechanism (named transitions, gates, signature groups) applies to Tasks, Cycles, and any future record types. The schema uses an open state pattern (`^[a-z][a-z0-9_]{0,49}$`) that accepts any state name.

This section defines the **Task lifecycle** as the primary use case (§7.1–7.9) and the **Cycle lifecycle** as a secondary example (§7.10). Because transition names are unique keys within `state_transitions`, a single workflow can govern multiple entity types by using distinct transition names (e.g. `submit` for tasks, `start_cycle` for cycles).

**The Task Lifecycle.** The protocol defines eight canonical states for the task lifecycle:

```
draft → review → ready → active → done → archived
                                ↘ paused ↗
                   review/ready/active → discarded
```

### 7.1. draft → review (Submission)

- **Transition name:** `submit`
- **Gate:** Command (`gitgov task submit`)
- **Condition:** Current status is `draft`.
- **Effect:** Status transitions to `review`.

### 7.2. review → ready (Plan Approval)

- **Transition name:** `approve`
- **Gate:** Command + Signature
- **Condition:** The signer has both the intent (`signature.role`) and the capability (`actor.roles`) to approve, per the Double Key Doctrine.
- **Effect:** Status transitions to `ready`.

### 7.3. ready → active (Activation)

- **Transition name:** `activate`
- **Gate:** Event (`first_execution_record_created`)
- **Condition 1:** Current status is `ready`.
- **Condition 2:** A valid assignment exists — a Feedback Record of `type: "assignment"` targeting this task exists, assigning the actor who creates the Execution Record (see §7.9).
- **Effect:** Status transitions to `active`.

### 7.4. active → done (Completion)

- **Transition name:** `complete`
- **Gate:** Command + Signature
- **Condition:** A valid `approver` signature per the methodology, from an actor with the required capability role (e.g. `approver:quality`).
- **Effect:** Status transitions to `done`.

### 7.5. done → archived (Formal Closure)

- **Transition name:** `archive`
- **Gate:** Command (`gitgov task archive`) + Signature
- **Condition:** The signer has both the intent and the capability to archive the task.
- **Effect:** Status transitions to `archived`.

### 7.6. active → paused (Suspension)

- **Transition name:** `pause`
- **Gate:** Event (`feedback_blocking_created`)
- **Condition:** A Feedback Record with `type: "blocking"` targeting this task has been created.
- **Effect:** Status transitions to `paused`.

### 7.7. paused → active (Resumption)

- **Transition name:** `resume`
- **Gate:** Command (`gitgov task resume`)
- **Condition:** All blocking Feedback Records on this task have been resolved.
- **Effect:** Status transitions to `active`.

> **Note:** `resume` is a distinct transition from `activate`. The named transition model allows different gates for the same target state: `activate` is event-driven (from `ready`), while `resume` is command-driven (from `paused`). This distinction is enforced by the schema — each transition has its own `from`, `to`, and `requires`.

### 7.8. * → discarded (Cancellation)

- **Transition name:** `cancel`
- **Gate:** Command (`gitgov task cancel`) + Signature
- **Condition:** The task is in `review`, `ready`, or `active`.
- **Effect:** Status transitions to `discarded`.

### 7.9. Assignment via Feedback Record

Task assignment is not a static field — it is a governable communication event managed through the Feedback Protocol (RFC-07).

To assign a task, an actor creates a Feedback Record with:
- `entityType`: `"task"`
- `entityId`: The task ID
- `type`: `"assignment"`
- `assignee`: The actor ID of the assignee

This Feedback Record becomes the auditable source of truth for responsibility. Reassignment is handled by creating a new assignment Feedback Record.

The `assignment_required` custom rule (§6.2) checks for the **existence** of an assignment Feedback Record targeting the task. The protocol does not mandate a specific assignment status — organizations may enforce acceptance via additional custom rules if needed.

### 7.10. Cycle Transitions

The same workflow mechanism governs Cycle Record (RFC-05) transitions. A workflow MAY include cycle transitions alongside task transitions. The four cycle states — `planning`, `active`, `completed`, `archived` — follow the same pattern:

```json
"start_cycle": {
  "from": ["planning"],
  "to": "active",
  "requires": {
    "command": "gitgov cycle start",
    "signatures": {
      "__default__": {
        "role": "author",
        "capability_roles": ["author"],
        "min_approvals": 1
      }
    }
  }
},
"complete_cycle": {
  "from": ["active"],
  "to": "completed",
  "requires": {
    "command": "gitgov cycle complete",
    "signatures": {
      "__default__": {
        "role": "approver",
        "capability_roles": ["approver:product"],
        "min_approvals": 1
      }
    }
  }
},
"archive_cycle": {
  "from": ["completed"],
  "to": "archived",
  "requires": {
    "command": "gitgov cycle archive"
  }
}
```

The `from`/`to` state names are scoped to each entity type — `active` in a task transition and `active` in a cycle transition are independent. The workflow engine resolves the correct transition by matching both the transition name and the entity's current state.

---

## 8. Custom States

The canonical states (8 for tasks, 4 for cycles) cover standard workflows. However, the schema uses an open pattern (`^[a-z][a-z0-9_]{0,49}$`) that allows custom states for any entity type.

This is particularly useful for:
- **Agent-driven workflows**: Intermediate states like `analyzing`, `planning`, `awaiting_approval`, or `executing`.
- **Domain-specific lifecycles**: Custom entity types (via `header.type: "custom"`) with their own state machines.
- **Extended workflows**: Additional states beyond the canonical ones for specialized processes.

The same Workflow engine, the same gate types, the same signature validation — applied to any state machine.

---

## 9. Agent Integration

A Workflow Record may declare which agents participate in the governance flow and when they activate. Agent details (engine, capabilities, knowledge) live in their Agent Record (RFC-03) — the workflow only defines the operational relationship.

| Field | Type | Required | Constraint | Description |
|:------|:-----|:---------|:-----------|:------------|
| `description` | string | No | maxLength: 200 | Brief description of the agent integration purpose. |
| `required_agents` | array | No | — | Array of agent entries, each identified by `id` or `required_roles` (§9.1) with `triggers` (§9.2). |

No additional properties are allowed on the `agent_integration` object.

### 9.1. Agent References

Each required agent is identified by either:
- **`id`**: Direct reference to a specific Agent Record (e.g. `agent:security-bot`). Pattern: `^agent(:[a-z0-9-]+)+$`.
- **`required_roles`**: Array of strings matching `^[a-z0-9-]+(:[a-z0-9-]+)*$`. At least one. Any agent with matching capability roles qualifies.

Each agent entry MUST include `triggers` (§9.2) and either `id` or `required_roles` (or both). No additional properties are allowed on agent entry objects.

### 9.2. Triggers

Each agent declares its `triggers` — the events that activate it:

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `event` | string | Yes | The event that activates the agent. |
| `action` | string | Yes | The action the agent should perform. |
| `cron` | string | No | Cron expression for scheduled triggers. |

No additional properties are allowed on trigger objects.

### 9.3. Example

```json
"agent_integration": {
  "description": "Automated security and review agents",
  "required_agents": [
    {
      "id": "agent:security-bot",
      "triggers": [
        {
          "event": "task_transitioned_to_review",
          "action": "run_security_scan"
        }
      ]
    },
    {
      "required_roles": ["reviewer:code"],
      "triggers": [
        {
          "event": "task_transitioned_to_review",
          "action": "auto_review"
        },
        {
          "event": "daily_standup",
          "action": "generate_summary",
          "cron": "0 9 * * 1-5"
        }
      ]
    }
  ]
}
```

---

## 10. Persistence

Workflow Records are persisted as JSON files wrapped in the Embedded Metadata Container (RFC-01):

```
.gitgov/workflows/<workflowId>.json
```

Examples:
- `.gitgov/workflows/1752270000-workflow-default.json`
- `.gitgov/workflows/1752350000-workflow-kanban.json`

---

## 11. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to WorkflowRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `INVALID_TRANSITION` | No transition matches the current state and requested action | `"No transition '{name}' from state '{from}'"` |
| `INSUFFICIENT_SIGNATURES` | Not enough valid signatures for a signature group | `"Signature group {group} requires {min} approvals, got {count}"` |
| `UNAUTHORIZED_SIGNER` | Signer lacks the required capability roles | `"Actor {keyId} does not have capability {role}"` |
| `CUSTOM_RULE_FAILED` | A custom rule validation did not pass | `"Custom rule {ruleId} failed: {reason}"` |

---

## 12. Examples

### 12.1. Simple Kanban

A minimal workflow for small teams with no signature requirements on most transitions:

```json
{
  "header": {
    "version": "1.0",
    "type": "workflow",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:team-lead",
        "role": "author",
        "notes": "Initial workflow configuration for the team",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752274500
      }
    ]
  },
  "payload": {
    "id": "1752270000-workflow-simple-kanban",
    "name": "Simple Kanban",
    "description": "Basic workflow for small teams",
    "state_transitions": {
      "submit": {
        "from": ["draft"],
        "to": "review",
        "requires": {
          "command": "gitgov task submit"
        }
      },
      "approve": {
        "from": ["review"],
        "to": "ready",
        "requires": {
          "command": "gitgov task approve",
          "signatures": {
            "__default__": {
              "role": "approver",
              "capability_roles": ["approver:product"],
              "min_approvals": 1
            }
          }
        }
      },
      "activate": {
        "from": ["ready"],
        "to": "active",
        "requires": {
          "event": "first_execution_record_created"
        }
      },
      "complete": {
        "from": ["active"],
        "to": "done",
        "requires": {
          "command": "gitgov task complete"
        }
      }
    }
  }
}
```

### 12.2. GitGovernance Default

The standard methodology with quality gates, agent integration, assignment validation, and separate `activate`/`resume` transitions:

```json
{
  "header": {
    "version": "1.0",
    "type": "workflow",
    "payloadChecksum": "b2c3d4e5f6a1789012345678901234567890123456789012345678901234efab",
    "signatures": [
      {
        "keyId": "human:protocol-lead",
        "role": "author",
        "notes": "GitGovernance default methodology with full quality gates",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752274600
      }
    ]
  },
  "payload": {
    "id": "1752274600-workflow-gitgovernance-default",
    "name": "GitGovernance Default Methodology",
    "description": "Standard GitGovernance workflow with quality gates and agent collaboration",
    "state_transitions": {
      "submit": {
        "from": ["draft"],
        "to": "review",
        "requires": {
          "command": "gitgov task submit",
          "signatures": {
            "__default__": {
              "role": "submitter",
              "capability_roles": ["author"],
              "min_approvals": 1
            }
          }
        }
      },
      "approve": {
        "from": ["review"],
        "to": "ready",
        "requires": {
          "command": "gitgov task approve",
          "signatures": {
            "__default__": {
              "role": "approver",
              "capability_roles": ["approver:product"],
              "min_approvals": 1
            },
            "design": {
              "role": "approver",
              "capability_roles": ["approver:design"],
              "min_approvals": 1
            },
            "quality": {
              "role": "approver",
              "capability_roles": ["approver:quality"],
              "min_approvals": 1
            }
          }
        }
      },
      "activate": {
        "from": ["ready"],
        "to": "active",
        "requires": {
          "event": "first_execution_record_created",
          "custom_rules": ["task_must_have_valid_assignment_for_executor"]
        }
      },
      "resume": {
        "from": ["paused"],
        "to": "active",
        "requires": {
          "command": "gitgov task resume"
        }
      },
      "complete": {
        "from": ["active"],
        "to": "done",
        "requires": {
          "command": "gitgov task complete",
          "signatures": {
            "__default__": {
              "role": "approver",
              "capability_roles": ["approver:quality"],
              "min_approvals": 1
            }
          }
        }
      },
      "archive": {
        "from": ["done"],
        "to": "archived",
        "requires": {
          "command": "gitgov task archive",
          "signatures": {
            "__default__": {
              "role": "approver",
              "capability_roles": ["approver:product"],
              "min_approvals": 1
            }
          }
        }
      },
      "pause": {
        "from": ["active"],
        "to": "paused",
        "requires": {
          "event": "feedback_blocking_created"
        }
      },
      "cancel": {
        "from": ["review", "ready", "active"],
        "to": "discarded",
        "requires": {
          "command": "gitgov task cancel",
          "signatures": {
            "__default__": {
              "role": "canceller",
              "capability_roles": ["approver:product", "approver:quality"],
              "min_approvals": 1
            }
          }
        }
      }
    },
    "custom_rules": {
      "task_must_have_valid_assignment_for_executor": {
        "description": "Task must have a valid assignment before execution can begin",
        "validation": "assignment_required"
      }
    }
  }
}
```

---

## 13. Security Considerations

**Double Key enforcement.** The combination of intent (signature role) and capability (actor roles) prevents both accidental and unauthorized state transitions. Neither key alone is sufficient.

**Fail-closed transitions.** If a transition is not explicitly defined in the active Workflow Record, it is rejected. The protocol does not allow implicit transitions. This is by design — transitions must be explicitly permitted.

**Custom rule isolation.** Custom rules with `expression` or `module_path` execute arbitrary logic. Implementations should sandbox these evaluations and treat their results as untrusted input that must not bypass signature requirements.

---

## 14. References

- Schema: `schemas/workflow_record_schema.yaml`
- Embedded Metadata: [RFC-01](./01_embedded.md)
- Actor Identity: [RFC-02](./02_actor.md)
- Agent (work contract, referenced in §9): [RFC-03](./03_agent.md)
- Task Record (primary lifecycle): [RFC-04](./04_task.md)
- Cycle Record (secondary lifecycle): [RFC-05](./05_cycle.md)
- Execution Record: [RFC-06](./06_execution.md)
- Feedback Record: [RFC-07](./07_feedback.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
