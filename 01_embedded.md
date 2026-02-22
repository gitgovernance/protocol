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

# RFC-01: Embedded Metadata

> Version: 1.0 | Status: Stable\
> Created: May 2025 | Last updated: 2026-02-04\
> Schema: `schemas/embedded_metadata_schema.yaml`

## Abstract

Embedded Metadata is the universal envelope for all GitGovernance records. It wraps any payload with a cryptographic header that provides integrity verification (SHA-256 checksums), authorship attribution (Ed25519 signatures), and schema validation — making every record a self-verifiable, immutable artifact.

---

## 1. Motivation

Every record in GitGovernance — a task, an execution, an actor identity — needs to answer three questions before it can be trusted: *Has it been tampered with? Who signed it? Is it structurally valid?*

The Embedded Metadata envelope answers all three by separating the domain data (payload) from its security metadata (header). This separation allows indexers and validators to verify integrity and authorship without parsing the domain-specific content, and enables decentralized verification without relying on any central authority.

---

## 2. Terminology

| Term | Definition |
|:-----|:-----------|
| **Record** | Any GitGovernance data artifact (task, actor, execution, etc.) wrapped in the Embedded Metadata envelope. |
| **Header** | The security layer containing version, type, checksum, and signatures. |
| **Payload** | The domain data whose structure is determined by `header.type`. |
| **Canonicalization** | The process of serializing the payload into a deterministic string for hashing. |
| **Signing Digest** | A composite string binding content, actor, role, time, and notes into a single signable value. |
| **Three Gates** | The three-step verification process: Integrity, Schema, Authentication. |

---

## 3. Record Structure

Every GitGovernance file conforms to this root structure:

```json
{
  "header": {
    "version": "1.0",
    "type": "<record_type>",
    "payloadChecksum": "<sha256_hex>",
    "signatures": [ ... ]
  },
  "payload": { ... }
}
```

The `header` can be parsed by any indexer or validator independently of the `payload`. The `payload` structure is determined by the schema corresponding to `header.type`.

No additional properties are allowed at the root level.

---

## 4. Header Specification

### 4.1. Required Fields

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `version` | string | `"1.0"` | Protocol version. |
| `type` | enum | `actor` &#124; `agent` &#124; `task` &#124; `execution` &#124; `feedback` &#124; `cycle` &#124; `workflow` &#124; `custom` | Determines which schema validates the payload. |
| `payloadChecksum` | string | `^[a-fA-F0-9]{64}$` | SHA-256 hash of the canonicalized payload (see §5.1). |
| `signatures` | array | `minItems: 1` | One or more Signature objects (see §4.3). |

### 4.2. Conditional Fields

These fields are required when `type` is `"custom"`:

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `schemaUrl` | string | string (expected to be a valid URL) | Public URL to the JSON Schema for the custom payload. |
| `schemaChecksum` | string | `^[a-fA-F0-9]{64}$` | SHA-256 of the schema file at `schemaUrl`, for integrity verification. |

No additional properties are allowed on the header object.

### 4.3. The Signature Object

Each entry in the `signatures` array has the following fields. No additional properties are allowed.

| Field | Type | Constraint | Description |
|:------|:-----|:-----------|:------------|
| `keyId` | string | `^(human\|agent)(:[a-z0-9-]+)+$` | The `id` of the signing ActorRecord. Supports scoped identifiers (e.g. `agent:camilo:cursor`). |
| `role` | string | `^([a-z-]+\|custom:[a-z0-9-]+)$`, 1–50 chars | The context role of the signature. Standard roles: `author`, `reviewer`, `approver`, `auditor`, `witness`, `collaborator`. Custom roles use the `custom:` prefix. |
| `timestamp` | integer | `>= 1600000000` | Unix timestamp (seconds) of when the signature was created. |
| `notes` | string | 1–1000 chars | Human-readable intent or justification. This field is mandatory — every signature must explain its purpose. Notes are part of the signing digest and are cryptographically protected. |
| `signature` | string | `^[A-Za-z0-9+/]{86}==$` | Ed25519 signature encoded in base64 (88 characters including padding). |

The `keyId` pattern uses `(:[a-z0-9-]+)+` (one or more segments) to support hierarchical scoping. A simple actor is `human:alice`. A scoped agent is `agent:camilo:cursor`. Both are valid.

---

## 5. Integrity Algorithms

### 5.1. Payload Canonicalization

To produce consistent checksums across languages and parsers, the payload is canonicalized before hashing. The algorithm follows the JSON Canonicalization Scheme (JCS, [RFC 8785](https://www.rfc-editor.org/rfc/rfc8785)):

1. **Sort keys** alphabetically (recursive, all levels).
2. **Serialize** to compact JSON (no whitespace between keys/values).
3. **Encode** the resulting string as UTF-8 bytes.

> **Note:** This protocol uses the subset of JCS relevant to its data types (strings, integers, booleans, arrays, objects, null). Floating-point serialization rules from JCS §3.2.2.3 do not apply — no GitGovernance schema uses floating-point fields.

```
payloadChecksum = SHA256(canonicalize(payload)).toHex()
```

The output is a 64-character lowercase hexadecimal string.

### 5.2. The Signing Digest

Signing only the `payloadChecksum` would prove that an actor certified the content, but not in what capacity, when, or why. To bind all five elements into a single cryptographic proof, the signature is computed over a composite digest:

```
digest = payloadChecksum + ":" + keyId + ":" + role + ":" + notes + ":" + timestamp
```

The signature is then:

```
signature = Ed25519_Sign(privateKey, SHA256(digest))
```

This prevents a valid `reviewer` signature from being reattached as an `approver`, stops notes from being modified after signing, and makes each signature non-replayable across different timestamps or records.

---

## 6. Verification: The Three Gates

A record is valid if and only if it passes all three gates in sequence:

### 6.1. Integrity

Re-canonicalize the `payload` and compute its SHA-256 hash. The record is valid only if:

```
SHA256(canonicalize(payload)) === header.payloadChecksum
```

### 6.2. Schema

Validate the `payload` against the JSON Schema corresponding to `header.type`. If the type is `custom`, the schema is fetched from `header.schemaUrl` and its integrity is verified against `header.schemaChecksum`.

### 6.3. Authentication

For each signature in `header.signatures`:

1. Retrieve the ActorRecord identified by `keyId`. If the actor's `status` is `"revoked"`, the signature is invalid — revoked keys MUST NOT be accepted for new operations (see RFC-02 §11).
2. Extract the actor's `publicKey`.
3. Reconstruct the signing digest: `payloadChecksum + ":" + keyId + ":" + role + ":" + notes + ":" + timestamp`.
4. Verify: `Ed25519_Verify(publicKey, SHA256(digest), signature)`.

If any signature fails verification, the entire record is invalid.

---

## 7. Persistence

All GitGovernance records are persisted as JSON files within the `.gitgov/` directory of a Git repository. The specific subdirectory is determined by `header.type`:

```
.gitgov/<type_plural>/<recordId>.json
```

Each record type RFC defines its canonical storage path. See RFC-02 (actors), RFC-04 (tasks), RFC-06 (executions), RFC-08 (workflows).

---

## 8. Error Codes

| Code | Condition | Message |
|:-----|:----------|:--------|
| `INVALID_SCHEMA` | Payload fails schema validation | `"Field {field}: {reason}"` |
| `CHECKSUM_MISMATCH` | Computed checksum does not match `header.payloadChecksum` | `"Payload checksum mismatch"` |
| `INVALID_SIGNATURE` | An Ed25519 signature fails verification | `"Signature for keyId {keyId} is invalid"` |
| `ACTOR_NOT_FOUND` | The `keyId` in a signature does not match any known ActorRecord | `"Actor {keyId} not found"` |

---

## 9. Examples

### 9.1. ActorRecord with a single signature

A new developer self-registers their identity:

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
    "signatures": [
      {
        "keyId": "human:lead-dev",
        "role": "author",
        "notes": "Self-registration of lead developer account",
        "signature": "lO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFgHiJkL8A==",
        "timestamp": 1752274500
      }
    ]
  },
  "payload": {
    "id": "human:lead-dev",
    "type": "human",
    "displayName": "Lead Developer",
    "publicKey": "MCowBQYDK2VwAyEAaBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789ABCDEF",
    "roles": ["developer", "reviewer"],
    "status": "active"
  }
}
```

### 9.2. ExecutionRecord with multiple signatures (scoped agent)

An agent creates a progress record, then a human reviews it:

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
        "notes": "OAuth 2.0 flow completed with GitHub provider integration",
        "signature": "kL8AlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyZaBcDeFA==",
        "timestamp": 1752274600
      },
      {
        "keyId": "human:camilo",
        "role": "reviewer",
        "notes": "Reviewed and tested locally. LGTM.",
        "signature": "mN9BpQ0zTlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXyA==",
        "timestamp": 1752274650
      }
    ]
  },
  "payload": {
    "id": "1752274600-exec-implement-oauth",
    "taskId": "1752274500-task-implement-oauth",
    "type": "progress",
    "title": "OAuth 2.0 flow implemented",
    "result": "Completed the OAuth 2.0 authentication flow with GitHub provider. Token refresh and session management included."
  }
}
```

The scoped `keyId` `agent:camilo:cursor` identifies both the owner (`camilo`) and the tool (`cursor`), enabling fine-grained attribution in hybrid human-AI workflows.

### 9.3. Custom record type

When extending the protocol with domain-specific records, `type: "custom"` requires a schema URL and its checksum:

```json
{
  "header": {
    "version": "1.0",
    "type": "custom",
    "schemaUrl": "https://example.com/schemas/deployment-record-v1.json",
    "schemaChecksum": "d4e5f6a1b2c3789012345678901234567890123456789012345678901234ijkl",
    "payloadChecksum": "e5f6a1b2c3d4789012345678901234567890123456789012345678901234mnop",
    "signatures": [
      {
        "keyId": "agent:deploy-bot",
        "role": "author",
        "notes": "Production deployment of v2.1.0",
        "signature": "nO0CqR1dUlO/ySk4mR8nT2vXwZq1pBcDfGhJkLmNoPqRsTuVwXyZaBcDeFgHiJkLmNoPqRsTuVwXA==",
        "timestamp": 1752274700
      }
    ]
  },
  "payload": {
    "deploymentId": "deploy-2025-07-12-v2.1.0",
    "environment": "production",
    "status": "success"
  }
}
```

---

## 10. Test Vectors

The following test vector allows implementers to verify their canonicalization, hashing, and signing logic against known-good values.

### 10.1. Key Material

| Field | Value |
|:------|:------|
| Seed (hex) | `cf9d1f40a38201304037ba03271ffbf19d0f77fa8cf1af8c81c9c7b8c26f5c08` |
| Public Key (base64) | `3vc0uW5wPsjZIBE+4FTk12eV0GsNh6B+DkDKMVsBZfU=` |
| Public Key (hex) | `def734b96e703ec8d920113ee054e4d76795d06b0d87a07e0e40ca315b0165f5` |

The seed is `SHA-256("gitgovernance-protocol-test-vector-seed-01")`. The Ed25519 keypair is derived from the first 32 bytes of this seed using standard PKCS#8 encoding. The public key is the raw 32-byte Ed25519 key (not SPKI-wrapped).

### 10.2. Payload (ActorRecord)

```json
{
  "id": "human:test-vector",
  "type": "human",
  "displayName": "Test Vector Actor",
  "publicKey": "3vc0uW5wPsjZIBE+4FTk12eV0GsNh6B+DkDKMVsBZfU=",
  "roles": ["developer"],
  "status": "active"
}
```

### 10.3. Canonicalization

After applying JCS (§5.1 — sort keys, compact JSON, UTF-8):

```
{"displayName":"Test Vector Actor","id":"human:test-vector","publicKey":"3vc0uW5wPsjZIBE+4FTk12eV0GsNh6B+DkDKMVsBZfU=","roles":["developer"],"status":"active","type":"human"}
```

### 10.4. Payload Checksum

```
payloadChecksum = SHA-256(canonicalized) = 9c2daedb6eca2316ab4045febcdf4698bae9cb48caba8367be22975d23537e2e
```

### 10.5. Signing Digest

| Field | Value |
|:------|:------|
| keyId | `human:test-vector` |
| role | `author` |
| notes | `Self-registration of test vector actor` |
| timestamp | `1700000000` |

```
digest = "9c2daedb6eca2316ab4045febcdf4698bae9cb48caba8367be22975d23537e2e:human:test-vector:author:Self-registration of test vector actor:1700000000"
```

```
SHA-256(digest) = 7414eb299379c5ed67380a267c7798078d23a13ec379a68809120fb9dcf9b493
```

### 10.6. Signature

```
Ed25519_Sign(privateKey, SHA-256(digest)) = IQWDS9+oWbpkmb0ASdQ9Z+f3WRVV3oyiKw3xsbMFIquBp8Jqs3K1E1E28NPA6BIjLPMPD+/ZolnaTMpP+DkaCg==
```

### 10.7. Complete Record

```json
{
  "header": {
    "version": "1.0",
    "type": "actor",
    "payloadChecksum": "9c2daedb6eca2316ab4045febcdf4698bae9cb48caba8367be22975d23537e2e",
    "signatures": [
      {
        "keyId": "human:test-vector",
        "role": "author",
        "notes": "Self-registration of test vector actor",
        "signature": "IQWDS9+oWbpkmb0ASdQ9Z+f3WRVV3oyiKw3xsbMFIquBp8Jqs3K1E1E28NPA6BIjLPMPD+/ZolnaTMpP+DkaCg==",
        "timestamp": 1700000000
      }
    ]
  },
  "payload": {
    "id": "human:test-vector",
    "type": "human",
    "displayName": "Test Vector Actor",
    "publicKey": "3vc0uW5wPsjZIBE+4FTk12eV0GsNh6B+DkDKMVsBZfU=",
    "roles": ["developer"],
    "status": "active"
  }
}
```

To verify: reconstruct the signing digest from the header fields, compute `SHA-256(digest)`, and call `Ed25519_Verify(publicKey, hash, signature)`. The result must be `true`.

Verified against two independent implementations: pure Node.js `crypto` module and the `@gitgov/core` reference SDK.

---

## 11. Security Considerations

**Immutability.** Once a record is signed, any modification to the payload invalidates the checksum, and any modification to the signature context (role, timestamp, notes) invalidates the signature. There is no mechanism to silently alter a signed record.

**Digest binding.** The signing digest includes the signer's identity, role, timestamp, and notes. This prevents signature reuse across contexts — a reviewer signature cannot be repurposed as an approver signature, even for the same payload.

**Key management.** Private keys are stored client-side and never appear in the repository. Public keys are distributed through ActorRecords (see RFC-02). Key rotation is handled through the succession mechanism defined in RFC-02.

**Distributed verification.** Because records are stored as files in a Git repository, any auditor with repository access can independently verify integrity and signatures without trusting a central server. This supports compliance scenarios where independent audit is required.

**Delimiter safety.** The signing digest uses `:` as a field separator, and the `notes` field may contain colon characters. This is not a security concern because the digest is never parsed or split — it is only ever _constructed_ from discrete fields. Both the signer and verifier build the identical string from the same five individual values (`payloadChecksum`, `keyId`, `role`, `notes`, `timestamp`), then hash and sign/verify the result. No implementation should attempt to recover field values by splitting the digest string.

**Minimum signatures.** Every record requires at least one signature. Anonymous or unsigned records are not valid in the protocol.

---

## 12. References

- Schema: `schemas/embedded_metadata_schema.yaml`
- Actor Identity: [RFC-02](./02_actor.md)
- Task: [RFC-04](./04_task.md)
- Execution: [RFC-06](./06_execution.md)
- Workflow: [RFC-08](./08_workflow.md)

---

## License

Copyright 2025-2026 [GitGovernance](https://www.gitgovernance.com)

This specification is part of the GitGovernance Protocol, designed and maintained by GitGovernance. Licensed under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0).
