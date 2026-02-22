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

# RFC-02: Actor Record

> Version: 1.0 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-04\
> Schema: `schemas/actor_record_schema.yaml`

## Abstract

The Actor Record is the identity layer of GitGovernance. It serves as the **Root of Trust** for the entire ecosystem. Unlike centralized identity providers (IAM, LDAP), GitGovernance identity is decentralized: each Actor is identified by a cryptographically signed record containing their Ed25519 public key. This allows the system to verify actions (signatures) completely offline, using only the repository content.

Every participant — human or AI agent — must have an Actor Record before they can create, sign, or approve any protocol record.

---

## 1. Motivation

A governance protocol that cannot verify *who* performed an action is fundamentally broken. In hybrid human-AI environments, the identity problem becomes more complex: the same team may include humans, autonomous agents, and locally-operated AI assistants — each with different levels of authority.

GitGovernance solves this with a single identity model:

- **Humans and agents share the same record structure.** The `type` field distinguishes them, but the cryptographic verification is identical.
- **Identity is hierarchical.** The ID format models ownership relationships — `agent:camilo:cursor` clearly states that the agent `cursor` is operated by the human `camilo`.
- **Identity is immutable.** Once created, an Actor Record's identity fields cannot be changed. Key rotation and role changes require creating a new Actor Record through the succession mechanism.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Actor Record** | A signed, immutable record representing a participant in the system. |
| **Actor ID** | A hierarchical, human-readable identifier following the pattern `{type}:{name}[:{scope}]*`. |
| **Capability Role** | A static permission in the Actor Record's `roles` array — what the actor *can* do. |
| **Context Role** | A dynamic intent in a signature's `role` field — what the actor *did* in a specific action. |
| **Succession** | The process of replacing an Actor Record with a new one (new keys, new roles) while maintaining an auditable chain. |
| **Root of Trust** | The first Actor Record in a repository, which bootstraps the chain of trust. |

---

## 3. Actor ID Structure

Every Actor MUST have a globally unique identifier following the hierarchical format:

```
{type}:{name}[:{scope}]*
```

- **`type`**: `human` or `agent`.
- **`name`**: The handle of the entity.
- **`scope`**: (Optional, repeatable) Namespace for local or delegated agents.

The full ID must match the pattern: `^(human|agent)(:[a-z0-9-]+)+$`

### 3.1. ID Examples

| ID | Meaning |
|:---|:--------|
| `human:camilo` | A human individual. Root entity. |
| `agent:aion` | An autonomous, cloud-based agent. Root entity. |
| `agent:camilo:cursor` | The agent `cursor` operated by the human `camilo`. Models "pair programming". |
| `agent:camilo:cursor:planner` | A sub-agent `planner` invoked by the agent `cursor`. The chain reflects the delegation hierarchy. |

---

## 4. Record Schema

The Actor Record is the `payload` inside an Embedded Metadata envelope (RFC-01) where `header.type` is `"actor"`. The schema is defined in `actor_record_schema.yaml`.

### 4.1. Mandatory Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `id` | string | `^(human\|agent)(:[a-z0-9-]+)+$` | Unique, human-readable identifier (see §3). |
| `type` | enum | `human` \| `agent` | The nature of the entity. |
| `displayName` | string | 1–100 characters | Human-readable name for UIs (e.g. "Alice Smith"). |
| `publicKey` | string | Base64, exactly 44 characters | The Ed25519 public key (32 bytes, base64-encoded). |
| `roles` | array of strings | pattern: `^[a-z0-9-]+(:[a-z0-9-]+)*$`, minItems: 1, uniqueItems | **Capability Roles**: hierarchical permissions. |

### 4.2. Optional Fields

| Field | Type | Constraint | Default | Description |
|:------|:-----|:-----------|:--------|:------------|
| `status` | enum | `active` \| `revoked` | `active` | Lifecycle state of the actor. |
| `supersededBy` | string | `^(human\|agent)(:[a-z0-9-]+)+$` | — | ID of the successor Actor (used when `status` is `revoked`). |
| `metadata` | object | Free-form JSON | — | Additional context for the actor's identity (see §4.3). |

No additional properties are allowed at the root level.

### 4.3. The `metadata` Field

The `metadata` field is an optional, open object designed for extensibility. It adds rich context to an actor's identity without polluting the core schema.

**For an agent:**

```json
"metadata": {
  "version": "2.1.3",
  "provider": "anthropic",
  "model": "claude-3-opus",
  "source": "https://github.com/company/agents/my-reviewer.md"
}
```

**For a human:**

```json
"metadata": {
  "team": "backend",
  "github_handle": "@camilo-gitgov"
}
```

**For a revoked actor (documenting reason):**

```json
"metadata": {
  "revocationReason": "Key rotation after security audit"
}
```

### 4.4. Capability Roles

Roles use a hierarchical format separated by colons. This allows both broad and fine-grained permissions:

| Role | Meaning |
|:-----|:--------|
| `admin:system` | Full system administration capability. |
| `developer:backend` | Backend development capability. |
| `developer:backend:go` | Specifically Go backend development. |
| `approver:quality` | Quality approval capability. |
| `planner:ai` | AI-driven planning capability. |
| `auditor` | Audit capability (no sub-hierarchy). |

---

## 5. Key Management

### 5.1. Algorithm

All keys MUST be **Ed25519** (EdDSA over Curve25519).

- **Public Key Size**: 32 bytes (encoded as exactly 44 Base64 characters).
- **Private Key Size**: 32 bytes (or 64 bytes extended).

### 5.2. Private Key Storage

Protocol implementations MUST NOT store private keys in the repository.

- **Recommendation**: Store in `.gitgov/actors/{actorId}.key`, added to `.gitignore`.
- **Security**: File permissions MUST be restricted to `0600` (owner read/write only).

The public key is stored in the Actor Record's `publicKey` field and committed to the repository. The private key is managed entirely by the actor's local environment.

---

## 6. Governance Protocols

### 6.1. Capability Roles vs. Context Roles

This distinction is fundamental to the Double Key Doctrine (RFC-08) and a common source of errors:

- **Capability Roles** (in `ActorRecord.roles`): Static permissions — what the actor *can* do.
  - Examples: `admin:system`, `developer:backend`, `approver:quality`
- **Context Roles** (in `Signature.role`): Dynamic intent — what the actor *did* in a specific action.
  - Examples: `author`, `submitter`, `approver`, `canceller`

**Rule**: An Actor can sign with a Context Role (e.g. `approver`) ONLY IF they possess a matching Capability Role (e.g. `approver:quality`) in their Actor Record. This enforcement is the Workflow's responsibility (RFC-08, §5).

### 6.2. Trust Bootstrapping

Who signs the first Actor?

1. **Genesis**: The first Actor (project creator) creates a self-signed Actor Record. This record becomes the **Root of Trust** for the repository.
2. **Chain of Trust**: All subsequent Actor Records MUST be signed by an existing Actor with `admin` capability (e.g. `admin:system`).

This creates a verifiable chain of trust from the genesis actor to every participant.

### 6.3. Key Rotation (Succession)

Keys cannot be changed in an immutable record. Rotation is achieved via **Succession**:

1. **Create the successor**: Generate a new Actor Record (e.g. `human:camilo_v2`) with a new key pair and roles.
2. **Link the chain**: The new record MUST include `metadata.replacesActorId` referencing the old actor ID.
3. **Revoke the predecessor**: Update the old Actor Record — set `status` to `"revoked"` and `supersededBy` to the new actor's ID.
4. **Sign with the old key**: Both the successor creation and the predecessor revocation MUST be signed with the **old key** (proof of ownership).

This process creates an immutable, auditable chain of identity succession.

### 6.4. The `replacesActorId` Convention

When a new Actor Record is created as a successor, it MUST include in its `metadata`:

```json
"metadata": {
  "replacesActorId": "human:camilo"
}
```

This field is the forward pointer in the succession chain. Combined with the predecessor's `supersededBy` field, it creates a bidirectional, verifiable link between the old and new identity.

---

## 7. Persistence

Actor Records are persisted as JSON files wrapped in the Embedded Metadata Container (RFC-01):

```
.gitgov/actors/<actorId>.json
```

Examples:
- `.gitgov/actors/human:camilo.json`
- `.gitgov/actors/agent:aion.json`
- `.gitgov/actors/agent:camilo:cursor.json`

---

## 8. Verification

Integrity of an Actor Record is verified through three checks:

1. **Payload Checksum**: The SHA-256 checksum of the payload matches `header.payloadChecksum`.
2. **Self-Consistency**: The `payload.id` matches the filename convention and the `keyId` used in signatures.
3. **Endorsement**: For non-genesis actors, at least one creation signature must belong to a valid actor with `admin` capability.

---

## 9. Error Codes

Common error codes (`INVALID_SCHEMA`, `CHECKSUM_MISMATCH`, `INVALID_SIGNATURE`, `ACTOR_NOT_FOUND`) are defined in RFC-01 and apply to all record types. The following error codes are specific to ActorRecord operations:

| Code | Condition | Message |
|:-----|:----------|:--------|
| `ACTOR_REVOKED` | Actor's status is `revoked` — cannot sign new records | `"Actor {actorId} is revoked; use successor {supersededBy}"` |
| `INSUFFICIENT_CAPABILITY` | Actor lacks the required capability role for the context role | `"Actor {actorId} does not have capability for role {contextRole}"` |
| `INVALID_PUBLIC_KEY` | Public key is not a valid Ed25519 base64 key (44 chars) | `"Invalid public key format for actor {actorId}"` |

---

## 10. Examples

### 10.1. Human Actor (Founder / Root of Trust)

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Genesis actor — root of trust for the repository",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752274500
      }
    ]
  },
  "payload": {
    "id": "human:camilo",
    "type": "human",
    "displayName": "Camilo (Founder)",
    "publicKey": "MCowBQYDK2VwAyEA8nQ7RpK2mT4vLxYzAbCdEfGhIjKlMnOpQrStUvWx",
    "roles": ["admin:system", "developer:backend", "approver:product"],
    "status": "active",
    "metadata": {
      "team": "founders",
      "github_handle": "@camilo-gitgov"
    }
  }
}
```

### 10.2. Autonomous Agent

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "b2c3d4e5f6a1789012345678901234567890123456789012345678901234efgh",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering Aion as autonomous planning agent",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752347700
      }
    ]
  },
  "payload": {
    "id": "agent:aion",
    "type": "agent",
    "displayName": "Aion - Autonomous Planner",
    "publicKey": "MCowBQYDK2VwAyEAzX9YpK3mT5vLxYzAbCdEfGhIjKlMnOpQrStUvWxYz",
    "roles": ["planner:ai", "executor:autonomous"],
    "status": "active",
    "metadata": {
      "version": "2.1.0",
      "provider": "anthropic",
      "model": "claude-3-opus",
      "source": "https://github.com/gitgov/agents/aion.md"
    }
  }
}
```

### 10.3. Local Agent with Scope (Pair Programming)

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "c3d4e5f6a1b2789012345678901234567890123456789012345678901234ijkl",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Registering Cursor as Camilo's local AI assistant",
        "signature": "Zq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyA==",
        "timestamp": 1752448900
      }
    ]
  },
  "payload": {
    "id": "agent:camilo:cursor",
    "type": "agent",
    "displayName": "Cursor AI (Camilo's Instance)",
    "publicKey": "MCowBQYDK2VwAyEAqR3TpK4mT6vLxYzAbCdEfGhIjKlMnOpQrStUvWxYa",
    "roles": ["developer:ai:paired"],
    "status": "active",
    "metadata": {
      "operator": "human:camilo",
      "environment": "local",
      "ide": "cursor"
    }
  }
}
```

### 10.4. Revoked Actor with Successor

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234mnop",
    "signatures": [
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Genesis actor — root of trust for the repository",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752274500
      },
      {
        "keyId": "human:camilo",
        "role": "author",
        "notes": "Revoking after key rotation — successor is human:camilo_v2",
        "signature": "PqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPA==",
        "timestamp": 1752550000
      }
    ]
  },
  "payload": {
    "id": "human:camilo",
    "type": "human",
    "displayName": "Camilo (Founder)",
    "publicKey": "MCowBQYDK2VwAyEA8nQ7RpK2mT4vLxYzAbCdEfGhIjKlMnOpQrStUvWx",
    "roles": ["admin:system", "developer:backend", "approver:product"],
    "status": "revoked",
    "supersededBy": "human:camilo_v2",
    "metadata": {
      "revocationReason": "Key rotation after security audit"
    }
  }
}
```

---

## 11. Security Considerations

**Private key isolation.** Private keys MUST never be committed to the repository. They are stored locally with restrictive file permissions and excluded via `.gitignore`.

**Revocation is permanent.** Once an Actor's status is `revoked`, their key MUST NOT be accepted for new operations. Historical signatures remain valid as a forensic record but carry no forward authority.

**Trust chain integrity.** Every non-genesis Actor Record must be endorsed by an existing admin. Breaking this chain (e.g. accepting an unsigned Actor) compromises the entire identity model.

**Single key per record.** Each Actor Record is bound to exactly one public key. Key rotation requires succession, not in-place mutation. This prevents key-reuse attacks and maintains a clean audit trail.

---

## 12. References

- Schema: `schemas/actor_record_schema.yaml`   
- Embedded Metadata: [RFC-01](./01_embedded.md)   
- Workflow (Double Key Doctrine): [RFC-08](./08_workflow.md)   

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
