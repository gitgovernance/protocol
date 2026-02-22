<!--
  Copyright 2025-2026 GitGovernance (https://www.gitgovernance.com)
  SPDX-License-Identifier: Apache-2.0
-->

# Contributing to the GitGovernance Protocol

Thank you for your interest in improving the protocol. This document explains how to propose changes.

## Reporting Issues

Use [GitHub Issues](https://github.com/gitgovernance/protocol/issues) to report errors, ambiguities, or missing information in the RFCs or schemas.

## Pull Requests

Pull requests are welcome for:

- **Typos and clarifications** — Fixing errors, improving wording, adding examples.
- **New examples** — Additional test vectors, record examples, or usage patterns.
- **Schema improvements** — Stricter validation, better descriptions, new optional fields.

## RFC Amendments

Changes to the normative content of an RFC (required fields, validation rules, algorithms) require a formal proposal:

1. Open an issue with the label `rfc-amendment`.
2. Describe the current behavior, the proposed change, and the motivation.
3. Include the corresponding schema change if applicable.
4. The change will be discussed before any PR is merged.

## New RFCs

To propose a new record type:

1. Open an issue with the label `new-rfc`.
2. Include an abstract, motivation, field definitions, and a draft JSON Schema.
3. New RFCs are assigned the next available number and must follow the existing format.

## Schema Coherence

Every change must maintain the **derivability principle**: from any RFC, you can derive exactly its schema, and from any schema, you can trace back to its RFC. A PR that modifies an RFC must include the corresponding schema update (and vice versa).

## Licensing

All contributions are licensed under [Apache-2.0](./LICENSE). By submitting a pull request, you agree that your contribution is provided under the same license.
