---
name: analyse-codebase
description: Generate a text-based Intermediate Representation (IR) of TypeScript business logic for migration to Rust WASM.
argument-hint: [typescript-source] [ir-output-path]
---

# Codebase Analysis Skill

## Overview

Analyze a TypeScript codebase to produce a reconstruction-grade Intermediate Representation (IR) containing domain-level business logic. The IR enables accurate migration to Rust WASM components.

---

## Arguments

1. **TypeScript Source** (`$ARGUMENTS[0]`): Path to the TypeScript source codebase
2. **IR Output Path** (`$ARGUMENTS[1]`): Output path for the IR file (e.g., `./ir/component.ir.md`)

---

## Required Permissions

⚠️ **RECOMMENDED**: For seamless execution, request `required_permissions: ["all"]` on the first Shell command to avoid multiple permission prompts during file analysis.

---

## Principles (Non-Negotiable)

1. **Focus**: Extract only domain/business logic and its inputs/outputs. Exclude infrastructure unless part of a domain rule.
2. **Descriptive, not interpretive**: Produce algorithmic descriptions of what the code does. Do not infer "why" unless present in source.
3. **Zero inference**: Do not invent behavior or semantics. Use explicit `unknown` tokens.
4. **Explicit constants**: List every constant by identifier and semantic availability.
5. **Traceability**: Each statement must be traceable to code. Do not attribute intent not in comments.
6. **Tagging**: Each Business Logic line must include one tag: `[domain]`, `[infrastructure]`, `[mechanical]`, or `[unknown]`.
7. **Conservatism**: Prefer `unknown` over guessing.

---

## Tags

| Tag | Description |
|-----|-------------|
| `[domain]` | Business logic semantics explicitly domain-related |
| `[infrastructure]` | Operational code visible in source |
| `[mechanical]` | Deterministic computation visible in source |
| `[unknown]` | Behavior or meaning cannot be deduced |

---

## Unknown Tokens (Use Exactly)

- `unknown — not present in source`
- `unknown — referenced symbol defined outside repo`
- `unknown — ambiguous control flow`
- `unknown — semantics not documented in source comments or code`

---

## Process

### Step 1: Identify Component Structure

Read the TypeScript source at `$ARGUMENTS[0]` and identify:

1. Entry points (`main.ts`, `index.ts`, handler exports)
2. Module organization and file structure
3. External dependencies from `package.json`
4. Async boundaries (`async/await`, Promises)
5. Type definitions (interfaces, types, classes)

### Step 2: Extract Business Logic

For each function/method, document:

- Symbol name and return type
- Algorithm (pseudocode with tags)
- Preconditions and postconditions
- Edge cases and failure modes
- Complexity/cost notes
- Inputs (with constants referenced)
- Outputs and errors
- Unknowns

### Step 3: Document External API Surfaces

For each external HTTP/API call:

- Endpoint URL pattern
- HTTP method
- Request headers (list each)
- Request body shape (exact JSON/XML structure)
- Response body shape (CRITICAL: capture full nesting)
- Authentication method
- Error responses (status codes and body shapes)

### Step 4: Capture Publication & Timing Patterns

Document exactly:

- Publication count (e.g., "twice with 5s delay")
- Timing/delay operations with exact durations
- Retry patterns with counts and backoff
- Batch vs individual publication

### Step 5: Write IR Output

Write the IR to `$ARGUMENTS[1]` using the exact format below.

---

## IR Output Format (Exact Structure)

```markdown
### Component

- Name: <component_name>
- Source paths: <list of source files>
- Purpose summary: <1-2 sentence description>

### Structures

- Imports: <list>
- Exports: <list>
- Types: <list with definitions>
- Classes: <list with key methods>
- Functions: <list with signatures>
- External dependencies: <from package.json>

### Business Logic Blocks (repeat per function/method)

- Symbol: <function_name>
- Return type: <type>
- [TAG] Algorithm (pseudocode):
  1) <step>
  2) <step>
- [TAG] Preconditions: <list>
- [TAG] Postconditions: <list>
- [TAG] Edge cases & failure modes: <list>
- [TAG] Complexity / cost notes: <notes>
- Inputs:
  - `<name>` — constant referenced; semantics: `<description>`
- Outputs: <description>
- Errors: <list>
- Unknowns: <list with tokens>

### External API Surfaces (repeat per external call)

- Endpoint URL pattern: <pattern>
- HTTP Method: <GET/POST/etc>
- Request Headers (list each): <headers>
- Request Body Shape: <exact JSON/XML structure>
- Response Body Shape: <exact structure - preserve full nesting>
- Authentication Method: <Bearer token/API key/none>
- Error Responses: <status codes and body shapes>

### Publication & Timing Patterns

- Publication count: <description>
- Timing/delay operations: <durations>
- Retry patterns: <counts and backoff>
- Batch vs individual: <description>

### Output Event Structures (preserve exact nesting)

<Document full nested structure for each output type>

### Dependencies

- `<package>` — `<purpose>`

### Notes

- <any additional observations>
```

---

## Verification Checklist

Before completing, verify:

- [ ] Every HTTP call has response shape documented with example
- [ ] Every external service call has authentication method noted
- [ ] Publication counts and timing delays are explicitly captured
- [ ] Output event nesting is preserved (not flattened)
- [ ] All `unknown` tokens use exact format
- [ ] Every business logic block has all required fields
- [ ] Tags are applied to every algorithm/precondition/postcondition line

---

## Output

Write the completed IR to `$ARGUMENTS[1]`. This file will be used by the `generate-crate` skill.

---

## Important Notes

- Do not simplify or interpret behavior beyond what is explicitly in the source
- Preserve exact field names and nesting structures
- When in doubt, use `unknown` tokens rather than guessing
- Focus on reconstruction-grade accuracy for downstream code generation
