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

# RFC-03: Agent Record

> Version: 1.1 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-20\
> Schema: `schemas/agent_record_schema.yaml`

## Abstract

The Agent Record is the **operational manifest** for AI agents in GitGovernance. While the Actor Record (RFC-02) defines an agent's *identity* — its public key, display name, and capability roles — the Agent Record defines its *function*: how to invoke it, what triggers activate it, and what knowledge it requires. This separation allows identity (who signs) and operations (what runs) to evolve independently.

The Agent Record achieves **implementation sovereignty**: it can describe agents built with any framework (LangChain, Google ADK, custom scripts), hosted anywhere (local, cloud, Docker), and communicating via any protocol (HTTP, MCP, A2A). The `engine` field is the universal adapter that makes this possible.

---

## 1. Motivation

In hybrid human-AI teams, agents come from diverse sources — local TypeScript functions, cloud-hosted APIs, MCP servers, IDE extensions, and emerging protocols like A2A. A governance protocol must integrate all of these without requiring the agent to "know" about the governance system.

GitGovernance solves this with three principles:

- **Identity and function are separate.** The Actor Record is the passport (who the agent *is*). The Agent Record is the work contract (what the agent *does*). The same identity can change its engine, triggers, or knowledge without a new key pair.
- **The engine is a universal adapter.** Four engine variants — `local`, `api`, `mcp`, `custom` — cover the full spectrum from native code to external services to future protocols. The protocol is agnostic to what runs behind the engine.
- **Schema-valid does not mean executable.** An Agent Record can exist in a minimal "stub" state (only `id` and `engine.type`) for drafting or planning. Execution validation happens at invocation time, not at record creation.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Agent Record** | A signed, immutable record defining the operational manifest of an AI agent. |
| **Agent ID** | The agent's identifier, which MUST match a corresponding ActorRecord of `type: "agent"`. Uses the same hierarchical format as Actor IDs. |
| **Engine** | The invocation specification — how to call the agent (local function, HTTP endpoint, MCP server, or custom protocol). |
| **Trigger** | An event definition that activates the agent (manual command, webhook event, or cron schedule). |
| **Knowledge Dependency** | A glob pattern specifying files the agent needs loaded into its context. |
| **Stub Agent** | An Agent Record that is schema-valid but not executable — used for drafts, placeholders, or disabled agents. |

---

## 3. Agent ID Structure

The Agent ID reuses the Actor ID format from RFC-02. Every Agent Record MUST have an `id` that corresponds 1:1 to an existing ActorRecord of `type: "agent"`.

**Pattern**: `^agent(:[a-z0-9-]+)+$`

### 3.1. ID Examples

| ID | Meaning |
|:---|:--------|
| `agent:scribe` | An autonomous documentation agent. |
| `agent:aion` | An autonomous cloud-based planning agent. |
| `agent:camilo:cursor` | The agent `cursor` operated by the human `camilo`. |
| `agent:deepl-translator` | A wrapper around an external translation API. |

### 3.2. Relationship with ActorRecord

The ActorRecord defines the agent's **identity** (public key, capability roles). The AgentRecord defines its **function** (engine, triggers, knowledge). They share the same `id` but serve different purposes:

| Aspect | ActorRecord (RFC-02) | AgentRecord (RFC-03) |
|:-------|:---------------------|:---------------------|
| Purpose | Identity and authorization | Invocation and operations |
| Key fields | `publicKey`, `roles`, `type` | `engine`, `triggers`, `knowledge_dependencies` |
| Analogy | Passport | Work contract |
| Mutable fields | None (succession required) | `engine`, `triggers`, `knowledge_dependencies`, `status` |

An AgentRecord CANNOT exist without a corresponding ActorRecord. The ActorRecord is created first; the AgentRecord extends it with operational configuration.

---

## 4. Record Schema

The Agent Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"agent"`. The schema is defined in `agent_record_schema.yaml`.

### 4.1. Mandatory Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `id` | string | `^agent(:[a-z0-9-]+)+$` | Agent identifier, linking 1:1 to an ActorRecord of `type: "agent"` (§3). |
| `engine` | object | oneOf: `local` \| `api` \| `mcp` \| `custom` | Invocation specification (§5). |

### 4.2. Optional Fields

| Field | Type | Constraint | Default | Description |
|:------|:-----|:-----------|:--------|:------------|
| `status` | enum | `active` \| `archived` | `active` | Operational status (§7.2). |
| `triggers` | array of objects | items require `type` field | `[]` | Events that activate the agent (§6). |
| `knowledge_dependencies` | array of strings | glob patterns | `[]` | Files the agent needs loaded into its context. |
| `prompt_engine_requirements` | object | `roles` and `skills` arrays | — | Requirements for prompt composition. |
| `metadata` | object | Free-form JSON | — | Framework-specific or deployment-specific information (§4.3). |

No additional properties are allowed at the root level.

### 4.3. The `metadata` Field

The `metadata` field is an optional, open object for extensibility. It does NOT affect agent execution — it is purely informational.

Common use cases:

- **Framework identification**: `{ "framework": "langchain", "version": "0.2.0" }`
- **Deployment info**: `{ "deployment": { "provider": "gcp", "region": "us-central1" } }`
- **Cost tracking**: `{ "cost_per_invocation": 0.03, "currency": "USD" }`
- **Tool capabilities**: `{ "accepts_tools": ["review", "refactor", "test"] }`

---

## 5. The Engine Object

The `engine` field defines how the agent is invoked. It uses a `oneOf` constraint with 4 mutually exclusive variants. Each variant has `additionalProperties: false` — fields from one variant cannot be mixed with another.

### 5.1. Variant: `local`

For agents that run as native code within the repository.

| Field | Required | Type | Description |
|:------|:---------|:-----|:------------|
| `type` | Yes | `"local"` | Engine type identifier. |
| `runtime` | No | string | Runtime environment (`typescript`, `python`, etc.). |
| `entrypoint` | No | string | Path to the agent entry file. |
| `function` | No | string | Function name to invoke. |

**Note**: A stub agent with only `type: "local"` is schema-valid but not executable. At least `runtime` or `entrypoint` is needed for invocation.

### 5.2. Variant: `api`

For agents exposed as HTTP endpoints — external services, cloud functions, or third-party APIs.

| Field | Required | Type | Description |
|:------|:---------|:-----|:------------|
| `type` | Yes | `"api"` | Engine type identifier. |
| `url` | Yes | string (URI) | HTTP endpoint for the agent. |
| `method` | No | enum: `POST` \| `GET` \| `PUT` | HTTP method. Default: `POST`. |
| `auth` | No | object | Authentication configuration (§5.5). |

### 5.3. Variant: `mcp`

For agents that communicate via the Model Context Protocol (MCP). An MCP server can expose multiple tools; the `tool` field controls access granularity.

| Field | Required | Type | Description |
|:------|:---------|:-----|:------------|
| `type` | Yes | `"mcp"` | Engine type identifier. |
| `url` | Yes | string (URI) | MCP server endpoint. |
| `tool` | No | string | Specific MCP tool to invoke. If omitted, the agent has access to all tools on the server. |
| `auth` | No | object | Authentication configuration (§5.5). |

**Granularity guidance**: Use a single AgentRecord without `tool` when the server is a "toolbox" used dynamically. Use multiple AgentRecords (one per tool) when you need separate identities, permissions, or triggers per tool.

### 5.4. Variant: `custom`

For emerging or specialized protocols that do not fit the other categories.

| Field | Required | Type | Description |
|:------|:---------|:-----|:------------|
| `type` | Yes | `"custom"` | Engine type identifier. |
| `protocol` | No | string | Custom protocol identifier (e.g., `a2a`, `grpc`). |
| `config` | No | object | Protocol-specific configuration (free-form). |

**Note**: The `custom` variant is the extensibility point. When a new protocol (A2A, gRPC, WebSockets) matures, it can be adopted without changes to the core schema.

### 5.5. Authentication

The `auth` object is available in `api` and `mcp` variants. It supports flexible authentication with `additionalProperties: true` for extension.

| Field | Type | Description |
|:------|:-----|:------------|
| `type` | enum: `bearer` \| `oauth` \| `api-key` \| `actor-signature` | Authentication method. |
| `secret_key` | string | Reference to a secret in Secret Manager (for bearer/api-key/oauth). |
| `token` | string | Direct token value (not recommended for production). |

**`actor-signature`**: The recommended method for inter-agent communication. The system automatically signs requests using the agent's ActorRecord private key. The receiving agent verifies the signature using the sender's public key. This eliminates secret management and provides cryptographic attribution for every request.

### 5.6. Schema Validation vs Execution Validation

The protocol distinguishes between these two validation layers:

| Aspect | Schema Validation | Execution Validation |
|:-------|:------------------|:---------------------|
| When | At record creation/save | At agent invocation |
| What | Structural correctness | Operational readiness |
| Error | `INVALID_SCHEMA` | Engine-specific errors |

A stub agent (e.g., `{ "id": "agent:watcher", "engine": { "type": "local" } }`) passes schema validation but fails execution validation because it lacks the fields needed to actually run. This is by design — it allows draft agents, placeholders, and temporarily disabled agents to exist as valid records.

---

## 6. Triggers

The `triggers` array defines events that activate the agent. Each trigger object requires a `type` field and allows additional fields (`additionalProperties: true`) for extensibility.

### 6.1. Manual

Invoked explicitly by a user or orchestrator.

| Field | Type | Description |
|:------|:-----|:------------|
| `type` | `"manual"` | Trigger type. |
| `command` | string (optional) | Example CLI command for documentation. |

### 6.2. Webhook

Activated by system or external events.

| Field | Type | Description |
|:------|:-----|:------------|
| `type` | `"webhook"` | Trigger type. |
| `event` | string (optional) | Event identifier (e.g., `task.ready`, `feedback.created`, `git.push`). |
| `filter` | string (optional) | Condition filter (e.g., `priority:high`, `branch:main`). |

### 6.3. Scheduled

Executed on a recurring schedule.

| Field | Type | Description |
|:------|:-----|:------------|
| `type` | `"scheduled"` | Trigger type. |
| `cron` | string (optional) | Standard cron expression (e.g., `0 9 * * 1-5` = 9am weekdays). |

If `triggers` is empty or omitted, the agent can only be invoked programmatically.

---

## 7. Governance Protocols

### 7.1. Creation and Linking

- An AgentRecord can only be created if a corresponding ActorRecord with the same `id` and `type: "agent"` already exists.
- The default `status` is `active`.
- Creation MUST be signed by an actor with appropriate capability roles (e.g., `admin:agent-platform`).

### 7.2. States

The AgentRecord has a binary lifecycle:

| State | Meaning |
|:------|:--------|
| `active` | The agent is operational and can be invoked. |
| `archived` | The agent is retired and cannot be invoked. Preserved as historical record. |

Archiving an agent does not affect its ActorRecord — the identity remains valid for signature verification of historical records.

### 7.3. Mutability

The `id` is immutable. All other fields — `engine`, `triggers`, `knowledge_dependencies`, `prompt_engine_requirements`, `status`, `metadata` — can be modified.

Any modification produces a new `payloadChecksum` in the Embedded Metadata envelope, requiring a new signature. Previous versions are preserved in Git history.

This means you can:
- Migrate an agent from `local` to `api` without changing its identity.
- Add or remove triggers without creating a new agent.
- Archive and reactivate agents by changing `status`.

### 7.4. Key Rotation (Succession Chain)

When the corresponding ActorRecord rotates its keys via the succession mechanism (RFC-02 §6.3), the AgentRecord **maintains its original ID**. The system resolves the succession chain automatically to find the current active identity.

This provides:
- **Immutability**: Historical records never change.
- **Continuity**: The agent continues operating under the same ID.
- **Auditability**: The chain of succession is explicit and verifiable.

---

## 8. Persistence

Agent Records are persisted as JSON files wrapped in the Embedded Metadata envelope (RFC-01):

```
.gitgov/agents/<agentId>.json
```

Examples:
- `.gitgov/agents/agent:scribe.json`
- `.gitgov/agents/agent:camilo:cursor.json`

---

## 9. Verification

Integrity of an Agent Record is verified through three checks:

1. **Payload Checksum**: The SHA-256 checksum of the canonically serialized payload matches `header.payloadChecksum`.
2. **Schema Validation**: The payload conforms to `agent_record_schema.yaml`.
3. **Actor Binding**: The `id` references a valid, existing ActorRecord of `type: "agent"`.

---

## 10. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to AgentRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `ACTOR_TYPE_MISMATCH` | ActorRecord exists but is not of type `agent` | `"ActorRecord {id} is type '{type}', expected 'agent'"` |
| `AGENT_ARCHIVED` | Invocation attempted on an archived agent | `"Agent {id} is archived and cannot be invoked"` |
| `DUPLICATE_ID` | Agent ID already exists | `"Agent {id} already exists"` |

---

## 11. Examples

### 11.1. Local TypeScript Agent

A native documentation agent with knowledge dependencies and a manual trigger:

```json
{
  "header": {
    "version": "1.0",
    "type": "agent",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering scribe agent for automated documentation generation",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752274500
      }
    ]
  },
  "payload": {
    "id": "agent:scribe",
    "status": "active",
    "engine": {
      "type": "local",
      "runtime": "typescript",
      "entrypoint": "packages/agents/scribe/index.ts",
      "function": "runScribe"
    },
    "triggers": [
      { "type": "manual" }
    ],
    "knowledge_dependencies": [
      "packages/blueprints/**/*.md"
    ],
    "metadata": {
      "purpose": "documentation-generation",
      "maintainer": "team:platform"
    }
  }
}
```

### 11.2. External API Agent with Actor-Signature Auth

A cloud-hosted sentiment analyzer using actor-signature for inter-agent trust:

```json
{
  "header": {
    "version": "1.0",
    "type": "agent",
    "payloadChecksum": "b2c3d4e5f6a1789012345678901234567890123456789012345678901234efgh",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering sentiment analyzer agent — triggers on feedback creation",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752347700
      }
    ]
  },
  "payload": {
    "id": "agent:sentiment-analyzer",
    "status": "active",
    "engine": {
      "type": "api",
      "url": "http://sentiment-analyzer:8082/analyze",
      "method": "POST",
      "auth": {
        "type": "actor-signature"
      }
    },
    "triggers": [
      { "type": "webhook", "event": "feedback.created" }
    ],
    "metadata": {
      "framework": "google-adk",
      "version": "1.0.0",
      "model": "gemini-pro",
      "deployment": {
        "runtime": "docker",
        "image": "gitgov/sentiment-analyzer:v1.2.0"
      }
    }
  }
}
```

### 11.3. MCP Agent with Tool Restriction

A code reviewer exposed via MCP, restricted to a single tool:

```json
{
  "header": {
    "version": "1.0",
    "type": "agent",
    "payloadChecksum": "c3d4e5f6a1b2789012345678901234567890123456789012345678901234ijkl",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering Cursor code reviewer via MCP — triggers on task ready",
        "signature": "Zq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyA==",
        "timestamp": 1752448900
      }
    ]
  },
  "payload": {
    "id": "agent:cursor-reviewer",
    "status": "active",
    "engine": {
      "type": "mcp",
      "url": "http://localhost:8083/mcp",
      "tool": "code-review",
      "auth": {
        "type": "actor-signature"
      }
    },
    "triggers": [
      { "type": "webhook", "event": "task.status.ready" }
    ],
    "knowledge_dependencies": [
      "packages/**/*.ts",
      "packages/**/*.tsx"
    ],
    "metadata": {
      "ide": "cursor",
      "accepts_tools": ["review", "refactor", "test"]
    }
  }
}
```

### 11.4. Custom Protocol Agent (A2A)

A coordination agent using an emerging protocol via the `custom` engine:

```json
{
  "header": {
    "version": "1.0",
    "type": "agent",
    "payloadChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234mnop",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering A2A coordinator agent — scheduled every 4 hours",
        "signature": "PqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPA==",
        "timestamp": 1752550000
      }
    ]
  },
  "payload": {
    "id": "agent:coordinator",
    "status": "active",
    "engine": {
      "type": "custom",
      "protocol": "a2a",
      "config": {
        "endpoint": "https://agent-hub.gitgov.io/a2a",
        "version": "draft-2025-01",
        "capabilities": ["task-delegation", "status-sync", "feedback-loop"]
      }
    },
    "triggers": [
      { "type": "scheduled", "cron": "0 */4 * * *" }
    ],
    "metadata": {
      "purpose": "multi-agent-orchestration",
      "experimental": true
    }
  }
}
```

---

## 12. Security Considerations

**Actor binding.** Every AgentRecord is cryptographically bound to an ActorRecord. An agent cannot sign records or be invoked without a valid identity. Revoking the ActorRecord effectively disables the agent.

**Engine isolation.** The protocol defines the record structure, not the execution sandbox. Implementations SHOULD enforce isolation between agents — a `local` agent should not have access to another agent's knowledge dependencies or credentials.

**Secret management.** The `auth.secret_key` field references secrets by name, not by value. Implementations MUST NOT store secrets in the Agent Record itself. Direct `token` values in `auth` are supported but discouraged for production use.

**Actor-signature over shared secrets.** For inter-agent communication, `actor-signature` is recommended over bearer tokens because it provides cryptographic attribution, eliminates shared secret management, and enables revocation via the ActorRecord succession mechanism.

**Stub agents.** A schema-valid stub agent cannot be invoked (execution validation fails), but it exists as a signed record. Implementations should ensure that stub agents do not bypass governance gates — an archived or stub agent must not appear in operational queries.

---

## 13. References

- Schema: `schemas/agent_record_schema.yaml`
- Embedded Metadata: [RFC-01](./01_embedded.md)
- Actor Identity: [RFC-02](./02_actor.md)
- Task Record: [RFC-04](./04_task.md)
- Execution Record: [RFC-06](./06_execution.md)
- Feedback Record: [RFC-07](./07_feedback.md)
- Workflow (transition rules): [RFC-08](./08_workflow.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
