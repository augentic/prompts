# Codebase Analysis Task

## Objective

Analyze the legacy codebase at `{{LEGACY_CODE}}` to understand its structure, dependencies,
patterns, and identify areas that will need special attention during migration.

## Deep Business-Logic Text IR (Highly Descriptive, Strict, Unambiguous)

You are a Node.js + TypeScript extraction expert. Your mission is to produce a text-based
Intermediate Representation (IR) that contains reconstruction-grade,
domain-level business logic for one or more TypeScript/JavaScript source files.
The IR must be highly descriptive but never infer or guess semantics not explicitly present in the
inspected source or explicit source comments.

Legacy source code to analyze (full component - injected):
{{LEGACY_CODE}}

---

## Principles (non-negotiable)

1. Focus: extract only domain/business logic and its inputs/outputs.
    Exclude infrastructure and I/O specifics unless they are part of a
    domain rule and clearly visible in source.
2. Descriptive, not interpretive: produce rich, algorithmic
    descriptions of what the code does. Do not produce why unless intent
    is present in source.
3. Zero inference: do not invent behavior, semantics, or intent. Use
    explicit unknown tokens.
4. Explicit constants: list every constant referenced by exact
    identifier and its semantic availability.
5. Traceability & citation: each factual statement must be traceable to
    code. Do not attribute intent not in comments.
6. Tagging: each Business Logic line must include one tag: [domain],
    [infrastructure], [mechanical], or [unknown].
7. Conservatism: prefer unknown over guessing.
8. Depth requirement: every Business Logic block must include
    Algorithm, Preconditions, Postconditions, Edge Cases, Complexity
    notes, and Examples if present in source. If absent, mark unknown.
9. Deterministic format: use the exact structure so downstream tooling
    can parse.

## Tags

[domain] — Business logic semantics explicitly domain-related.
[infrastructure] — Operational code visible in source.
[mechanical] — Deterministic computation visible in source.
[unknown] — Behavior or meaning cannot be deduced.

## Unknown token (use exactly these)

unknown — not present in source
unknown — referenced symbol defined outside repo
unknown — ambiguous control flow
unknown — semantics not documented in source comments or code

## Output location

- Write final IR text to #file:{{IR_PATH}} (stringified plain text payload).
- This text IR replaces the JSON schema but must stay deterministic for tooling.

## Output format (Plain Text ONLY, EXACT ORDER)

### Component

- Name:
- Source paths:
- Purpose summary:

### Structures

- Imports:
- Exports:
- Types:
- Classes:
- Functions:
- External dependencies:

### Business Logic Blocks (repeat per function/method)

- Symbol:
- Return type:
- [TAG] Algorithm (pseudocode):
  1)
  2)
- [TAG] Preconditions:
- [TAG] Postconditions:
- [TAG] Edge cases & failure modes:
- [TAG] Complexity / cost notes:
- Inputs:
  - `<name>` — constant referenced; semantics: `<defined-in-source | unknown>`
- Outputs:
- Errors:
- Unknowns:

### External API Surfaces (repeat per external call)

- Endpoint URL pattern:
- HTTP Method:
- Request Headers (list each):
- Request Body Shape (exact JSON/XML structure if present):
- Response Body Shape (exact JSON/XML structure - CRITICAL: capture full nesting):
  Example: `[ "string1", "string2" ]` NOT `{ "items": [{ "label": "..." }] }`
- Authentication Method (e.g., Bearer token, API key, none):
- Error Responses (status codes and body shapes):

### Publication & Timing Patterns (CRITICAL - capture exactly)

- Publication count: How many times is each message published? (e.g., "twice with 5s delay")
- Timing/delay operations: Any sleep(), setTimeout(), or delay patterns with exact durations
- Retry patterns: Any retry loops with counts and backoff
- Batch vs individual: Are items published individually or batched?

### Output Event Structures (preserve exact nesting)

- For each output type, document the FULL nested structure:
  Example:

  ```rust
  SmarTrakEvent {
    receivedAt: DateTime,
    eventType: enum,
    messageData: {
      timestamp: DateTime,
      // ... nested fields
    },
    remoteData: {
      externalId: string,
      // ... nested fields
    }
  }
  ```

- Never flatten nested objects in IR - preserve hierarchy exactly as in source

### Dependencies

- `<dependency>` — `<semantics | unknown>`

### Notes

- <free-form observations, comment-claimed gaps, TODO markers>

## API Response Verification Checklist

- [ ] Every HTTP call has its response shape documented with example
- [ ] Every external service call has authentication method noted
- [ ] Publication counts and timing delays are explicitly captured
- [ ] Output event nesting is preserved (not flattened)

END OF TEXT IR (Do not append extra commentary)
