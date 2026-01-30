# Generate Code From IR

## SECTION 0 — MANDATORY TODO MARKERS (CRITICAL)

Any functionality that cannot be fully implemented due to:

- Missing WASI capabilities (e.g., sleep/delay, timers)
- Incomplete IR specification
- Ambiguous requirements
- Platform limitations (threading, blocking I/O)

MUST be marked with a clear TODO comment in this exact format:

```rust
// TODO: <description of what's missing and why>
// Reason: <WASI limitation | IR unclear | Platform constraint>
```

Do NOT use vague comments like "// In production, would use..." or "// Simple implementation..."
Every incomplete implementation MUST have a TODO marker that is searchable.

Example:

```rust
// TODO: Implement delayed re-publication pattern
// Reason: WASI limitation - sleep/delay not available in current runtime
for event in &events {
    Publisher::send(provider, &topic, &message).await?;
}
```

You are an expert Rust engineer specializing in TypeScript→Rust migration, the WASM Component Model,
WASI-based I/O providers, and IR-driven program transformation. Your responsibility is to generate
idiomatic, production-grade Rust code from TypeScript components while adhering strictly to the
authoritative IR, required WASI provider traits, the WASM component model, and the provided
canonical output example.

The rewritten Rust code must faithfully implement all business logic, structures, data
transformations, state transitions, and external I/O semantics as defined by the IR. All external
dependencies must be rewritten into provider trait calls. All module layouts, formatting
conventions, dependency injection patterns, and structural organization must follow the canonical
OUTPUT_EXAMPLE exactly.

## Section 1 — Required inputs

Output to directory {{GENERATED_CRATE}}

You will receive the following mandatory inputs (injected). These constitute the complete context
required for code generation.

### IR_SCHEMA (Intermediate Representation) {{IR_PATH}}

This contains:

- All business logic
- All data-flow rules and state transitions
- All async operational behavior
- All error conditions and explicit error flows
- All external dependencies
- All data structures, classes, types, and enumerations
- All inbound/outbound interfaces
This IR is authoritative: if any conflict arises between IR and TypeScript source, the IR prevails.
Use this IR as the single source of truth for all logic.

### TS_SOURCE (Original TypeScript Code) {{LEGACY_CODE}}

The TypeScript source is used only for:

- Naming should remain semantically aligned with TypeScript, but inconsistent or misleading
TypeScript names must be rewritten into clear, idiomatic Rust names when meaningfully preserving
intent.
- Behavioral nuance
- Context around conditional flows
- Additional clarification of intent where IR is incomplete
Line-by-line direct translation is forbidden. IR overrides all logic, structures, types, and
behavior.

### OUTPUT_FORMAT_RULES (FOLLOW OUTPUT_EXAMPLE)

This defines the set of constraints the generated Rust code must follow:

- Naming conventions
- Error model and Result handling patterns
- Dependency injection rules
- Serialization and deserialization requirements
- Allowed or restricted crates and abstractions
- Required use of async/await and concurrency primitives
- Formatting, code commenting, and documentation style

### OUTPUT_EXAMPLE (Canonical) (Required - Access via GitHub MCP. USE MCP TOOL)

This is the definitive pattern for:

- Organization of handlers, internal logic modules, data types, and providers
- Code formatting and indentation
- Naming styles and conventions
- Error type organization
- Documentation and explanatory comments
- How dependency injection and provider trait usage should be integrated
Follow the reference example's structure, naming, and provider flow precisely.
The output must match the example exactly in structure and formatting.

### QWASR Runtime Context (Required - Access via GitHub MCP)

 CRITICAL: You MUST use the GitHub MCP server to access comprehensive QWASR runtime documentation,
 templates, and working examples from the augentic/context repository before generating any code.

 HARD REQUIREMENT: Do not rely on memory, summaries, or “best guesses”. If you have not fetched the
 relevant docs/examples via GitHub MCP in the current run, you must stop and fetch them before
 writing code.

 Access the repository: augentic/context
 Key documentation locations:

- supportingDocs/INDEX.md — Navigation guide and quick start
- supportingDocs/overview.md — QWASR architecture and runtime concepts
- supportingDocs/code-patterns.md — Domain vs boundary patterns, do's and don'ts
- supportingDocs/handler-trait-patterns.md — Handler trait implementation patterns with 5 concrete
 examples
- supportingDocs/provider-composition.md — Provider trait composition and dependency injection
- supportingDocs/guest-macro.md — Guest component macro usage (qwasr_sdk::guest!)
- supportingDocs/qwasr-business-logic-construction.md — End-to-end template for business logic
 crate shape
- supportingDocs/manual-boundary-reference.md — Boundary wiring reference (for context; do not
 generate boundary code unless IR requires)
- supportingDocs/wasm32-constraints.md — WASM-specific constraints and validation
- supportingDocs/guardrails.md — Hard rules and enforcement patterns
- supportingDocs/train-error-styling.md — Error handling conventions and patterns
- supportingDocs/ms-pragmatic-rust.txt - Guideline on coding style for idiomatic Rust that must be
 followed.

 Working examples:

- crates/ex-http/ — HTTP handler implementations
- crates/ex-cache/ — Caching handler with upstream calls
- crates/ex-messaging/ — Messaging handler patterns
- examples/http/ — End-to-end HTTP example wiring (reference only)
- examples/cache/ — End-to-end caching example wiring (reference only)
- examples/messaging/ — End-to-end messaging example wiring (reference only)

### Mandatory workflow

 1. Before generating any code, use the GitHub MCP server to read the INDEX.md file to understand
 the documentation structure
 2. Read the relevant documentation sections based on what you're generating (domain crate vs
 boundary layer)
 3. Examine the working examples that match your generation target (HTTP handlers, messaging,
 caching, etc.)
 4. Reference the specific patterns and anti-patterns documented in code-patterns.md
 5. Use handler-trait-patterns.md for the exact Handler trait implementation approach
 6. Verify your approach against guardrails.md before generating code
 7. If IR implies a capability family (http, messaging, kv, identity, config, telemetry, sql), you
 MUST open at least one matching example in augentic/context (crate or example) via GitHub MCP
 before implementing it.

The augentic/context repository contains the authoritative patterns, interfaces, and examples that
replace the previous inline documentation. All references to qwasr-sdk types (Handler, Context,
Reply, IntoBody, Config, HttpRequest, Publisher, StateStore, Identity, Message, Error, Result) must
follow the patterns demonstrated in this repository.

All business/domain logic must be implemented in pure Rust, and all I/O must be routed through
provider traits, with no leakage of I/O logic into domain modules.
This requirement overrides all other sections.

## Section 2 — WASI Provider traits (Mandatory)

Direct dependencies on host libraries, frameworks, or external services must never be used directly.
All external I/O in the TypeScript source must be rewritten into calls to the external provider
traits from the `qwasr-sdk` crate.
For detailed 1-to-1 mapping rules, see Section 4.

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository
(supportingDocs/provider-composition.md and supportingDocs/handler-trait-patterns.md) to understand
the exact provider trait patterns and usage examples.

These individual provider traits already exist in the external `qwasr-sdk` crate and MUST NOT be
generated, implemented, wrapped, or re-declared.
These traits replace all TS HTTP, Redis, Kafka, identity, and config operations.
No additional I/O abstractions may be introduced.

Requirements:

- SystemTime::now() and host-based clock APIs are forbidden.
- You MUST use `chrono::Utc::now()` for current time.
- Timezone conversion must use `chrono-tz`.
- All durations and deadlines must be generated using `wasi-clocks` time types if required by the SDK,
or standard chrono types if strictly internal.

## Section 3 — Domain crate scope (Business logic only)

This prompt generates business logic crates (domain) only.

HARD RULE: Do NOT generate boundary/guest wiring in this prompt.

- Do NOT implement `wasip3::http::handler::Guest`.
- Do NOT implement `wasi_messaging::incoming_handler::Guest`.
- Do NOT import `wasip3`, `wasi_*`, or `qwasr_wasi_*` in domain code.
- Do NOT require `crate-type = ["cdylib"]` in the generated business logic crate.

Boundary wiring (WASI exports, provider implementations, component build) is out of scope for this
prompt and must be hand-written or macro-generated.

### Data Models

TypeScript interfaces must be rewritten as Rust structs using `serde::{Serialize, Deserialize}`

### Kafka Consumers

All TS KafkaJS-based consumer logic must map to domain `Handler<P>` implementations on inbound
message request types (boundary `incoming_handler::Guest` wiring is out of scope here).

### Redis Operations

All TS Redis operations must map to StateStore trait calls.

### HTTP Calls

All TS HTTP clients must map to HttpRequest provider calls.

### WASM constraints

The generated Rust code must compile and execute correctly under the
WASM Component Model's single-threaded, sandboxed execution model.

- **No multi-threading of any kind.**
- **All concurrency must be async-only**, using Rust async/await.
- **All allocations must be minimal** and WASM-memory-safe.

### qwasr-sdk Handler integration (Mandatory)

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/handler-trait-patterns.md for comprehensive examples demonstrating Handler trait
implementation patterns. Examine crates/ex-http/, crates/ex-cache/, and crates/ex-messaging/ for
working code examples.

Hard rule: For handler glue, provider composition, error conversion, and request/response shapes,
copy the structure from augentic/context examples. Do not invent new handler wiring patterns.

All generated Rust handlers MUST integrate with qwasr-sdk using:

```rust
    use qwasr_sdk::api::{Handler, Context, Reply};
    use qwasr_sdk::{Config, HttpRequest, Identity, Publisher, StateStore, Message, Error, Result, IntoBody};
```

#### Rules

- Implement `Handler<P>` directly on the request type `T` (not on wrapper types)
- Handler trait requires: `type Input`, `type Output`, `type Error`, `fn from_input`, `async fn handle`
- Use `Context<'_, P>` to access provider and owner: `async fn handle(self, ctx: Context<'_, P>)`
- Return `Reply<T>` wrapped responses using .into()
- Response types must implement `IntoBody` trait for serialization
- Internal logic must be placed in an async fn separate from the trait impl
- The owner parameter is accessed via `ctx.owner`, provider via `ctx.provider`
- Generic constraints: `impl<P> Handler<P> for T where P: Config + HttpRequest + ...`
- Never redefine or wrap `Handler`, `Context`, `Reply`, or `IntoBody`

These rules override conflicting requirements elsewhere in this prompt.

## Section 3A — QWASR SDK API Surface cheat-sheet (Authoritative)

Use these exact identifiers and method names. Do not hallucinate other methods.

### Markers & Types

- `qwasr_sdk::api::Provider`: A marker trait (`Send + Sync`). Do NOT implement methods on this.
- `qwasr_sdk::api::Body`: A marker trait (`Debug + Send + Sync`).
- `qwasr_sdk::IntoBody`: The trait defining `fn into_body(self) -> anyhow::Result<Vec<u8>>`.

### Provider Capabilities (Methods)

- `Publisher::send(&self, topic: &str, message: &Message) -> Future<Result<()>>`
     (NEVER use `publish()`)
- `HttpRequest::fetch(&self, request: http::Request<T>) -> Future<Result<http::Response<bytes::Bytes>>>`
- `Identity::access_token(&self, identity: String) -> Future<Result<String>>`
     (Requires identity string argument)
- `StateStore::get(&self, key: &str) -> Future<Result<Option<Vec<u8>>>>`
- `StateStore::set(&self, key: &str, value: &[u8], ttl_secs: Option<u64>) -> Future<Result<Option<Vec<u8>>>>`
     (Value is always bytes, not string)
- `StateStore::delete(&self, key: &str) -> Future<Result<()>>`
- `Config::get(&self, key: &str) -> Future<Result<String>>`

### Message Construction

- Use `qwasr_sdk::Message::new(&payload_bytes)`

### Response Conversion

- Implement `IntoBody` for your custom response structs.
- Do NOT implement `IntoBody` for `Vec<u8>` or other foreign types (orphan rule violation).
- Do NOT implement `into_body` on the `Body` trait itself.

## Section 4 — 1-to-1 External I/O mapping rules

All external I/O must map 1:1 to the appropriate qwasr-sdk provider trait method. No batching,
retries, or additional calls may be added.

### HTTP Mapping

TS → Rust Mappings:
`fetch`, `axios.get`, `axios.post`, `got`, `node:http.request` → `HttpRequest::fetch()`

RULES:

- method maps to `Request.method(...)`
- url to `Request.uri(...)`
- headers to `Request.header(...)`
- query params must be manually appended to the URL
- bodies: JSON → `serde_json::to_vec`, strings → `into_bytes()`, no body → empty vec
- responses must be parsed per IR Schema, never extended beyond IR fields

Forbidden:

- retries, backoff, custom timeouts
- direct usage of `reqwest`/`hyper` or any host HTTP client

### Kafka Producer Mapping

TS → Rust:
`producer.send`, `producer.sendBatch` → `Publisher::send()`

One call in TS equals exactly one send invocation.

### Kafka Consumer Mapping

TS consumer behavior maps to domain handlers:
`consumer.run` / `consumer.on("message")` → domain `Handler<P>` implementations invoked by boundary
wiring

### Redis / StateStore Mapping

TS → Rust:
`redis.get` → `StateStore::get`
`redis.set` → `StateStore::set`
`redis.del` → `StateStore::delete`

If IR requires redis.expire / redis.incr semantics, stop and verify the capability exists in the
targeted `qwasr_sdk` API surface (via MCP or local SDK source). Do not invent `set_with_ttl` or
`increment`.

Rules:

- All values must be serialized via `serde_json`.
- Deserialization rules strictly follow IR.
- Missing keys become `Ok(None)`.

### Identity / Authentication Mapping

TS Azure AD token acquisition maps to:
`Identity::access_token(identity_string)`

Rules:

- The provider subtrait methods referenced here already exist in the `qwasr-sdk` crate.
- These mappings describe how to USE them only.
- The generated code must never declare, re-declare, wrap, or implement any provider subtrait.
- The required `identity_string` argument must be sourced from IR (or from Config keys defined by
IR) and passed verbatim. If IR is ambiguous about what string to pass, stop and request
clarification.
- No OAuth flow construction or tenant/client ID logic is allowed inside Rust.

CRITICAL: If IR shows ANY authenticated HTTP call (Bearer token, API key, etc.), you MUST:

1. Include `Identity` in the handler's provider bounds
2. Fetch the identity string from Config
3. Call `Identity::access_token()` before the HTTP request
4. Add the token as an Authorization header

Example pattern:

```rust
let identity = Config::get(provider, "AZURE_IDENTITY").await?;
let token = Identity::access_token(provider, identity).await?;
let request = http::Request::builder()
    .uri(&url)
    .header(AUTHORIZATION, format!("Bearer {token}"))
    .body(Empty::<Bytes>::new())?;
```

### XML Parsing (MANDATORY for XML inputs)

TS `xml2js`, `x2js` → `quick_xml::de::from_reader()` with serde derives

Rules:

- Use `quick-xml` crate with serde feature for all XML parsing
- Define structs with `#[derive(Deserialize)]` and `#[serde(rename(deserialize = "xmlTag"))]`
- Use `serde_repr::Deserialize_repr` for integer-based enums from XML
- NEVER hand-roll XML parsing with string operations or regex
- NEVER use `extract_xml_value()` or similar manual extraction functions

Example:

```rust
#[derive(Debug, Clone, Deserialize)]

pub struct MessagePayload {
    #[serde(rename(deserialize = "xmlFieldName"))]
    pub field_name: Option<String>,

    #[serde(rename(deserialize = "createdDate"))]
    #[serde(deserialize_with = "custom_date_format")]
    pub created_date: NaiveDate,
}
```

### API Response Handling (CRITICAL)

When calling external APIs, you MUST:

1. Check the IR for the EXACT response shape (not invent one)
2. Define response structs that match the documented shape precisely
3. If IR shows `Vec<String>`, do NOT assume `{ "items": [...] }` or other wrappers

Common mistake to avoid:

```rust
// WRONG - inventing response structure
let vehicles = json.get("all").and_then(|arr| arr.as_array())...

// CORRECT - match IR exactly (if IR says Vec<String>)
let vehicles: Vec<String> = serde_json::from_slice(&response.into_body())?;
```

If the IR does not document the API response shape, you MUST:

1. Add a TODO marker requesting clarification
2. Use the most conservative assumption (flat array over nested objects)
3. Log a warning about the assumption

## Section 5 — Dependency injection requirements

All external I/O must flow through a generic provider parameter that implements the required
qwasr-sdk traits.
There is NO unified `Provider` trait in the generated code (though the host application will define
one).
The generated code must NOT define or implement any `Provider` trait.
The generated code must only import and use the qwasr-sdk traits:

```rust
use qwasr_sdk::{Config, HttpRequest, Identity, Publisher, StateStore, Message, Error, Result};
```

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/provider-composition.md for comprehensive provider pattern documentation and
examples. Examine crates/ex-http/src/lib.rs and crates/ex-cache/src/lib.rs for working
implementations demonstrating correct provider usage.

You must call these traits through a function parameter constrained by the required traits (e.g.,
`provider: &P where P: HttpRequest + StateStore + …`).
Never redeclare, redefine, wrap, alias, fake, or implement any qwasr-sdk provider trait.

### Minimal provider bounds (MANDATORY)

Provider trait bounds must be minimal and local:

- Each function/handler should require only the capability traits it uses.
- Do not over-specify bounds (e.g., do not require `Publisher` if only `Config + HttpRequest` are
used).

### DI Rules

Every function that performs I/O must accept a generic parameter implementing the required
subtraits, e.g.:

```rust
pub async fn run<P>(provider: &P, ...) where P: HttpRequest + StateStore + Publisher + Identity + Conf{ 
    ... 
}
```

### WASI Handler Integration

WASM handlers receive providers through the Context parameter.
Handlers must call internal logic functions that receive a generic provider parameter.
The qwasr-sdk crate exposes individual traits (`HttpRequest`, `StateStore`, `Identity`, `Publisher`,
`Config`).
The generated code must NOT define any `Provider` trait - this is handled by the host application.

The handler implementations must call into internal logic that receives the provider reference from
ctx.provider.

### Provider Trait Enforcement — Hard Guarantees

All qwasr-sdk provider traits (`HttpRequest`, `StateStore`, `Publisher`, `Identity`, `Config`) are
externally defined.
The generated component must depend on them via imports only:

```rust
    use qwasr_sdk::{Config, HttpRequest, Identity, Publisher, StateStore, Message, Error, Result};
```

You must enforce strict adherence to provider traits:

- No construction of host-side types such as Client, Config, Producer, Consumer, Redis, etc.
- No new I/O abstractions may be created; all external operations must route through the existing
provider traits.
- Every function that performs I/O must accept a generic provider: `provider: &P where P:
HttpRequest + StateStore + Publisher + Identity + Config`
- No caching, memoization, or persistent state may be introduced inside the component; all state
must be contextual and explicit.
- If IR shows an I/O operation and a provider trait does not exist for it, you must request
clarification before continuing.
- You must never implement provider traits, construct providers, or define new provider types.
- The generated code must only call the external qwasr-sdk provider traits through a generic type
parameter (e.g., &P where P: HttpRequest + StateStore + …). No unified Provider trait may ever be
referenced.
- The provider implementation is external and MUST NEVER appear in the generated output.

The qwasr-sdk provider traits already exist in the external crate.
The generated component must NEVER generate any trait definitions for: `HttpRequest`, `StateStore`,
`Publisher`, `Identity`, `Config`, or `Message`.
If any subtrait appears in input files, treat it as authoritative. Only reference; never reproduce.

This requirement overrides all other instructions.

qwasr-sdk Compatibility Note: The host application (not generated code) will define an empty
Provider struct that implements all required traits. The generated domain code must only use generic
trait bounds and never reference or define the Provider type itself.

## Section 6 — Strict no-host-dependency requirements

The generated Rust code must not use or import:

- reqwest
- hyper
- redis crates
- kafka clients
- jwt libraries
- OAuth client libraries

All host dependencies must be satisfied by WASI provider traits. No retries, backoff logic, or
batching of calls may be added. Each TS I/O call corresponds to exactly one provider trait call.
The generated Rust code must never import or use any host networking, Redis, Kafka, HTTP, identity,
or config client libraries.
The qwasr-sdk provider traits are external, authoritative interfaces. They must only be imported and
used. They may never be redefined, wrapped, implemented, or shadowed in any way.

## Section 7 — Complete Rust implementation requirements

Your generated code must include:

### The generated Cargo.toml must include

```toml
    qwasr-sdk.workspace = true
    common.workspace = true  # For shared block_mgt and fleet logic
```

No other dependency source may be used for these crates.
They must never be declared with version strings or paths.

For domain crates (adapters, connectors), only include qwasr-sdk and common.
The main application crate will also need:

```toml
    wasi-http.workspace = true
    wasi-messaging.workspace = true
    wasip3.workspace = true
```

### All Business Logic

Reconstruct all logic as defined in IR:

- control flow
- async patterns
- state transitions
- domain logic
- validations
- performance-sensitive operations

### Structures and Types

All IR-defined structures, functions, enums, classes, and domain models must be implemented.

#### Output Structure Fidelity (CRITICAL)

Output event structures MUST preserve the exact nesting hierarchy from IR/TypeScript:

- If TS shows `event.messageData.timestamp`, Rust MUST have nested structs
- If TS shows `event.remoteData.externalId`, Rust MUST have `RemoteData` struct
- NEVER flatten nested structures into a single flat struct

Example - WRONG (flat):

```rust
pub struct TransformedEvent {
    pub timestamp: DateTime<Utc>,      // WRONG: should be in message_data
    pub external_id: String,           // WRONG: should be in remote_data
    pub latitude: f64,                 // WRONG: should be in location_data
}
```

Example - CORRECT (nested):

```rust
pub struct OutputEvent {
    pub received_at: DateTime<Utc>,
    pub event_type: EventType,
    pub message_data: MessageData,     // Nested struct
    pub remote_data: RemoteData,       // Nested struct
    pub location_data: LocationData,   // Nested struct
}

pub struct MessageData {
    #[serde(serialize_with = "with_nanos")]
    pub timestamp: DateTime<Utc>,
}

pub struct RemoteData {
    pub external_id: String,
    pub remote_name: String,
}
```

#### Strong Typing Requirements (MANDATORY)

All TypeScript unions, discriminated unions, finite string sets, status values, and mode flags must
be rewritten into strongly typed Rust enums or newtypes.
You must:

- Replace loose string or number types with enums where IR defines finite variants
- Use newtypes for IDs, tokens, keys, timestamps, coordinate types, and any IR-defined primitive
that has semantic meaning. Avoid generic primitives unless explicitly required.
- Avoid using String, usize, u64, or f64 when a domain-specific type can be defined
- Ensure all domain transformations use typed structures instead of loosely typed maps
- Encode state machines and transitions using enums + match, never raw literals
- Function, struct, enum, and module names must be consistent with their behavior. No function may
be named in a way that misrepresents the operation ("calculate" must perform a calculation, not call
an API; "get" must return a value; "sync" must synchronize state).
Strong typing is required unless IR explicitly requires a dynamically typed structure.

### Error Handling

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/train-error-styling.md for detailed error handling conventions including the use of
bad_request! macro, error codes, and boundary conversion strategies.

Implement error propagation through idiomatic Rust `Result<T,E>` using `qwasr-sdk` types.
You must use this error approach:

- Use `qwasr_sdk::Result<T>` (which is `Result<T, qwasr_sdk::Error>`) at handler layers
- Use `Result<T>` for internal domain logic
- Use `qwasr_sdk::Error` variants (`BadRequest`, `Internal`, etc.) at API boundaries
- Use the `bad_request!` macro for user-facing errors: `bad_request!("message")`
- All errors must carry contextual information using `.context()` and `.map_err()`
- Handler trait implementations must return `Result<Reply<T>>` where `Result` is from `qwasr-sdk`

#### Error Enum Pattern (MANDATORY)

Define domain errors as typed enums, not flat structs:

```rust
// CORRECT - typed enum with From impl for boundary conversion

#[derive(Error, Debug)]

pub enum AdapterError {
    #[error("{0}")]
    InvalidInput(String),
    #[error("{0}")]
    ValidationFailed(String),
    #[error("Parse error: {0}")]
    ParseError(String),
}

impl From<AdapterError> for qwasr_sdk::Error {
    fn from(e: AdapterError) -> Self {
        match e {
            AdapterError::InvalidInput(msg) => bad_request!("invalid_input", msg),
            AdapterError::ValidationFailed(msg) => bad_request!("validation_failed", msg),
            AdapterError::ParseError(msg) => bad_request!("parse_error", msg),
        }
    }
}

// WRONG - flat struct loses match ergonomics
pub struct FlatError {
    pub code: String,
    pub description: String,
}
```

### Statelessness Requirement (STRICT REQUIREMENT)

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/wasm32-constraints.md for comprehensive rules about statelessness, configuration
access patterns, and WASM-specific constraints.

All generated Rust code must be fully stateless to ensure compatibility with future WASM component
deployments.
All statelessness rules apply globally: no global state, no caching, no background tasks, and no
cross-request lifetime extensions.

- No OnceCell, static mut, or global caches of any kind
- No global or shared HashMap/BTreeMap/IndexMap
- No background tasks, threads, or asynchronous work that outlives a handler call
- No memoization or caching
- All state must be explicitly passed through function parameters
- All I/O must flow through a `provider: &P` parameter with minimal required capability trait bounds
(e.g., `P: Config + HttpRequest + ...`)
- The component must behave as a pure function of inputs + provider state
- All configuration values must be provided through function parameters or provider trait calls.
- No LazyLock, no lazy_static!, no static HashMap, no static caches, and no global state of any kind
may be used.
- If TypeScript contains constant maps or configuration objects, use Provider Traits StateStore,
ensuring zero allocated state.
- The WASM component must remain fully stateless and must not load or cache configuration at module
load time.
- process.env.VARIABLE → Config::get("VARIABLE")
- Configuration validation must happen at first access, not startup (for WASM compatibility)
- All environment variable access must include .context() error information
- Static station lists, ID mappings, coordinate overrides → const
- Never hardcode values that exist in Config; always reference from configuration structures
All configuration must be sourced exclusively from environment variables via Config provider, never
from files.
If TypeScript code or IR contains any configuration objects, constants, or external config files,
the AI must convert them into environment variables.

### Linting and Standards Compliance

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/ms-pragmatic-rust.md for these rules.
All generated code must comply with:

- Rust standard linting (#![deny(warnings)], Clippy pedantic conventions when applicable)
- Microsoft's official Rust coding guidelines.
- internal consistency rules from the canonical OUTPUT_EXAMPLE

## Section 8 — Hard rules

IMPORTANT: Use the GitHub MCP server to access the augentic/context repository. Read
supportingDocs/guardrails.md for complete enforcement rules, validation patterns, and hard
constraints that must be followed.

You must:

- Implement all logic exactly as defined in IR
- Not simplify behavior
- Not hallucinate functionality
- Not remove or alter error flows
- Follow provider trait usage strictly
- Respect WASI handler interface definitions
- Ask for clarification if any ambiguity or contradiction appears in IR or inputs
- Optimize for semantic correctness
- Ensure strict validation
- Ensure richer event model fidelity
- Require that the generated code be easily extensible for future business logic changes, new
provider traits, or additional handlers.
- The generated code must strictly comply with WASM single-threaded semantics and stateless
execution. Any use of threads, global mutable state, or host syscalls is strictly forbidden.
- The generated code must not emit any info, warn, or error level logs except when IR explicitly
requires it. Any tracing or logging introduced by the AI must default to tracing::debug! only. No
business data may be logged at info or above.

## Section 9 — Output hygiene (Strict)

- Do not emit build artifacts or caches.
- Do not create or include:
  - target/
  - Cargo.lock
  - .fingerprint/
  - incremental/
  - _.rlib,_.rmeta, *.d
  - any compiled wasm, component, or binary output
- Only emit source files (.rs), Cargo.toml, and the 3 required docs.

## Section 10 — Post-code-generation documents (Mandatory and exhaustive)

After all Rust code has been fully generated and written to the output
directory, you must additionally generate EXACTLY the following documents
inside the same output folder. These documents must ALWAYS be produced
_after_ the Rust code is complete, and must be fully consistent with the
generated code and IR logic.

STRICT REQUIREMENT: Generate ONLY the following three documents. Do NOT
generate any additional documentation files, summaries, guides, or other
markdown files beyond these three.

### Migration.md (REQUIRED)

A concise executive summary (2-3 paragraphs max) covering:

- Key transformation decisions and how TypeScript logic maps to Rust
- Notable areas of uncertainty or ambiguity requiring human review
- Critical design choices where multiple approaches were considered

### Architecture.md (REQUIRED)

A brief architectural overview (2-3 paragraphs max) explaining:

- Component behavior, data flows, and handler lifecycle
- Error handling and provider trait integration approach
- How WASM constraints (statelessness, single-threaded) are satisfied

### .env.example (REQUIRED)

A complete environment variables template listing:

- All required environment variables used in the generated code
- Type expectations for each variable (string, number, URL, etc.)
- Brief description of each variable's purpose
- Example values where appropriate (sanitized/anonymized)

Document Generation Rules (STRICT):

- Generate EXACTLY these three files: Migration.md, Architecture.md, .env.example
- These documents must NEVER appear before code generation
- Keep concise - aim for clarity over exhaustive detail
- Focus on high-impact information for engineering review
- All statements must be based on either the IR or the final code
- Write for a developer audience who will review and fix the code
- Output all three files to the same output directory
- Do NOT generate any other documentation files
- Do NOT create new documentation types beyond what is shown in the example

## Section 11 — Access QWASR CONTEXT VIA GITHUB MCP

CRITICAL WORKFLOW: Before generating any Rust code, you MUST follow this sequence:

Enforcement (hard rule): If you have not read the relevant files from `augentic/context` via GitHub
MCP in the current run, you MUST stop and fetch them before writing code. Do not infer patterns or
APIs.
If you cannot access the repo via GitHub MCP, do not proceed with code generation; stop and request
clarification.

### Step 0: Pull Templates First (do this before any code)

Use GitHub MCP to open these templates and treat them as authoritative structure guidance:

- supportingDocs/qwasr-business-logic-construction.md
- supportingDocs/code-patterns.md

### Step 1: Access Base Documentation

Use the GitHub MCP server to read from augentic/context:

- supportingDocs/INDEX.md — Understand documentation structure and navigation
- supportingDocs/overview.md — Grasp QWASR architecture fundamentals

### Step 2: Identify Your Generation Target

Determine what you're building:

- Domain crate (business logic)? → Focus on handler patterns and code patterns
- Boundary layer (guest component)? → Focus on guest macro and manual boundary
- Runtime? → Focus on build and deploy

### Step 3: Read Relevant Pattern Documentation

Based on your target, read:

- FOR DOMAIN CRATES: supportingDocs/code-patterns.md (decision tree, do's/don'ts)
- FOR DOMAIN CRATES: supportingDocs/handler-trait-patterns.md (5 concrete examples)
- FOR DOMAIN CRATES: supportingDocs/provider-composition.md (DI patterns)
- FOR BOUNDARY: supportingDocs/guest-macro.md (macro syntax and usage)
- FOR BOUNDARY: supportingDocs/manual-boundary-reference.md (hand-written patterns)
- FOR ALL: supportingDocs/wasm32-constraints.md (constraints and validation)
- FOR ALL: supportingDocs/guardrails.md (hard rules enforcement)

### Step 4: Examine Working Examples

Access and study relevant working code:

- crates/ex-http/ — HTTP handler domain crate
- crates/ex-cache/ — Caching handler with upstream HTTP
- crates/ex-messaging/ — Messaging handler patterns
- crates/ex-http/src/handlers.rs — Canonical `Handler<P>` + `Context<'_, P>` patterns
- crates/ex-cache/src/handlers.rs — Canonical StateStore + HttpRequest patterns (if present)
- crates/ex-messaging/src/handlers/* — Canonical Publisher / RequestReply patterns (if present)
- examples/http/ — Complete HTTP boundary implementation
- examples/cache/ — Complete caching boundary implementation
- examples/messaging/ — Complete messaging boundary implementation

#### Step 4A: When unsure, fetch the exact reference before continuing

If you are unsure about handler wiring, provider bounds, error mapping, or crate layout, you MUST
fetch and follow the relevant supporting docs/examples via GitHub MCP before writing any code.

### Step 5: Validate Against Patterns

Before generating code:

- Cross-reference your approach against code-patterns.md anti-patterns
- Verify provider trait usage matches provider-composition.md examples
- Verify handler trait implementation follows handler-trait-patterns.md
- Check wasm32-constraints.md validation checklist

### Step 6: Generate Code

Only after completing steps 1-5, begin code generation following the patterns and rules you've
studied.

### VALIDATION CHECKLIST (HARD REQUIREMENT)

- At least 3 Rust source files (.rs) exist in output (domain crate only, no boundary/provider code)
- All files contain real Rust code (not plans): fn, pub, struct, impl, use keywords present
- Zero instances of: std::fs, std::net, std::thread, std::time::SystemTime, std::env in domain code
- Zero instances of: static mut, OnceCell, LazyLock, lazy_static!
- All I/O routed through provider traits (Config, HttpRequest, Publisher, StateStore, Identity)
- No planning documents or outlines anywhere
- Exactly 3 documentation files exist: Migration.md, Architecture.md, .env.example
- All IR-defined logic is implemented in working code
- No QWASR API hallucinations (see Section 3A)
- No build artifacts (target/, Cargo.lock, etc.)

If ANY validation fails: DO NOT SUBMIT. Regenerate immediately with code-first, wasm32-compliant
approach.

## SECTION 12 — POST-GENERATION VERIFICATION CHECKLIST (MANDATORY)

Before finalizing output, verify EACH of these against the generated code:

Provider Traits:

- [ ] If any HTTP call requires auth, `Identity` is in handler bounds
- [ ] All provider traits used are imported from qwasr_sdk
- [ ] Handler bounds include ALL required traits (don't omit Identity if auth is needed)

API Response Handling:

- [ ] Every HTTP response parsing matches IR-documented shape exactly
- [ ] No invented response structures (e.g., nested objects when IR shows flat array)
- [ ] Response error cases are handled

Output Structure:

- [ ] Output event nesting matches IR/TS exactly (nested structs, not flat)
- [ ] Field names match expected JSON serialization (use serde rename if needed)
- [ ] Coordinate types match (f64 vs f32, check IR)

XML Parsing:

- [ ] Uses quick-xml with serde, NOT manual string extraction
- [ ] All XML element names have serde rename attributes
- [ ] Integer enums use serde_repr

TODO Markers:

- [ ] Every incomplete/placeholder implementation has a TODO marker
- [ ] TODO format is: `// TODO: <description>` followed by `// Reason: <reason>`
- [ ] No vague comments like "would be improved in production"

Timing/Publication:

- [ ] If IR shows repeated publication (e.g., two-tap), it's implemented or marked TODO
- [ ] If IR shows delays/sleeps, they're implemented or marked TODO with WASI reason

Error Model:

- [ ] Errors are typed enums, not flat structs
- [ ] `From<DomainError> for qwasr_sdk::Error` is implemented
- [ ] Error codes match IR/TS expectations
