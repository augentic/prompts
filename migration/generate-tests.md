# Augentic WASM Replayer Test Generation Prompt

You are an expert Rust developer working with the Augentic WASM runtime and testing framework. Your
task is to generate a comprehensive "Replayer" test harness for a specific WASM component. This
harness allows running the component against recorded production data (snapshots) to verify behavior
correctness, specifically handling time-dependent logic and external I/O mocking.

PLACES TO KNOW

## Injected inputs

- IR: {{IR_PATH}}
- Legacy source: {{LEGACY_CODE}}
- Output dir/ location of component: {{OUTPUT_DIR}}

## Goal

Construct a Rust test module that:

1. **Analyzes** the target component to identify its input/output types and external dependencies
(**MANDATORY: Use MCP to read component source code and dependencies**).
2. **Defines** a `Replay` struct implementing the `augentic_test::Fixture` trait to map JSON test
definitions to these types.
3. **Implements** a `MockProvider` to intercept and mock external calls (HTTP, PubSub, Config,
Identity) and capture outputs.
4. **Implements** a time-shifting transformation function to adjust historical data (timestamps) in
inputs to be relative to `Utc::now()`.
5. **Provides** a test runner that executes the component against a set of JSON data files and
performs deep verification.
6. **Documents** the test harness with a glossary, versioning information, and performance/security
notes.

## Context Gathering

**CRITICAL: Accurate context is essential. Without it, the generated test harness will be incorrect
and unusable.**

To generate accurate test harnesses, **MANDATORILY** gather comprehensive context from the target
component's repository using GitHub MCP access (or approved fallback if MCP is unavailable):

1. **Component Source Code**: Read the main library file (e.g., `src/lib.rs`) and key handler
functions to identify input/output types and external dependencies.
2. **Test Data Structure**: Examine existing test JSON files in `data/` or `tests/data/` to
understand the schema for inputs, params, http_requests, and outputs.
3. **Provider Traits**: Identify which `qwasr_sdk` traits the component uses (e.g., `HttpRequest`,
`Publisher`, `Config`, `Identity`).
4. **Time Logic**: Search for timestamp comparisons or `Utc::now()` usage to determine if
`shift_time` is required.
5. **Metadata**: Read `Cargo.toml` for dependencies and `README.md` for component description.

## Implementation Steps

### Phase 1: Context Analysis

Before writing code, **MANDATORILY use MCP to analyze the target component's GitHub repository**:

- **Input Type**: What struct does the component accept? (e.g., `R9kMessage`, `HttpRequest`) -
**Check via MCP**.
- **Output Type**: What does it produce? (e.g., `Vec<SmarTrakEvent>`) - **Verify with MCP**.
- **Dependencies**: Does it use `qwasr_sdk::Config`, `Publisher`, `Identity`? - **Inspect source
code via MCP**.
- **Time Sensitivity**: Does the component compare timestamps against "now"? If so, `shift_time` is
critical - **Search code with MCP**.

### Phase 2: Define the Replay Fixture

Create `tests/provider.rs`. Define the `Replay` struct implementing `augentic_test::Fixture`.

**CRITICAL**: You MUST implement the `augentic_test::Fixture` trait exactly as shown below. Do NOT
create your own `TestFixture` struct or custom trait.

#### EXACT Fixture Trait Definition (from augentic-test crate - DO NOT MODIFY)

```rust
// This is the EXACT trait from augentic-test - copy these associated types exactly
pub trait Fixture: Sized + Clone {
    type Error: Clone;
    type Input: Clone;
    type Output: Clone;
    type TransformParams: Clone + Default;

    fn from_data(data_def: &TestDef<Self::Error>) -> Self;
    fn input(&self) -> Option<Self::Input>;
    fn params(&self) -> Option<Self::TransformParams>;
    fn output(&self) -> Option<Result<Self::Output, Self::Error>>;
}
```

#### Required Imports

```rust
use augentic_test::{Fetcher, Fixture, PreparedTestCase, TestCase, TestDef, TestResult};
use qwasr_sdk::{api::Client, Config, HttpRequest, Identity, Publisher, Message, Error, Result};
```

#### Replay Struct Implementation

```rust
#[derive(Debug, Clone)]

pub struct Replay {
    pub input: Option<ComponentInputType>,  // The typed input (e.g., R9kMessage)
    pub params: Option<ReplayTransform>,
    pub output: Option<ReplayOutput>,
}

#[derive(Debug, Clone, Default, Deserialize)]

pub struct ReplayTransform {
    pub delay: i32,  // Seconds offset for time shifting
}

// CRITICAL: Output enum must match actual fixture JSON structure

#[derive(Debug, Clone, Deserialize)]
#[serde(untagged)]

pub enum ReplayOutput {
    Events(Vec<ExpectedOutputType>),  // For success cases
    Error(qwasr_sdk::Error),          // For failure cases
}

impl Fixture for Replay {
    type Error = qwasr_sdk::Error;
    type Input = ComponentInputType;  // e.g., R9kMessage
    type Output = Vec<ExpectedOutputType>;  // e.g., Vec<SmarTrakEvent>
    type TransformParams = ReplayTransform;

    fn from_data(data_def: &TestDef<Self::Error>) -> Self {
        // 1. Parse input (handle XML string -> typed struct if needed)
        let input = data_def.input.as_ref().map(|v| {
            let xml_str: String = serde_json::from_value(v.clone())
                .expect("input should be string");
            quick_xml::de::from_str(&xml_str)
                .expect("XML should parse to input type")
        });

        // 2. Parse params
        let params = data_def.params.as_ref().map(|v| {
            serde_json::from_value(v.clone()).expect("params should parse")
        });

        // 3. Parse output - handle TestResult enum
        let output = data_def.output.as_ref().map(|result| match result {
            TestResult::Success(v) => ReplayOutput::Events(
                serde_json::from_value(v.clone()).expect("events should parse")
            ),
            TestResult::Failure(e) => ReplayOutput::Error(e.clone()),
        });

        Self { input, params, output }
    }

    fn input(&self) -> Option<Self::Input> {
        self.input.clone()
    }

    fn params(&self) -> Option<Self::TransformParams> {
        self.params.clone()
    }

    fn output(&self) -> Option<Result<Self::Output, Self::Error>> {
        self.output.as_ref().map(|o| match o {
            ReplayOutput::Events(events) => Ok(events.clone()),
            ReplayOutput::Error(e) => Err(e.clone()),
        })
    }
}
```

### Phase 3: Implement the Mock Provider

The `MockProvider` must capture side-effects (published messages) and provide mocked responses.

**CRITICAL**:

1. Use `Fetcher` from `augentic_test` for HTTP mocking - do NOT implement manual request matching
2. Include ALL provider traits the component uses (especially `Identity` if auth is needed)
3. Use `Arc<Mutex<Vec<_>>>` for captured events to allow access after handler execution

```rust
#[derive(Clone)]

pub struct MockProvider {
    test_case: PreparedTestCase<Replay>,
    events: Arc<Mutex<Vec<ExpectedOutputType>>>,
}

impl MockProvider {
    pub fn new(test_case: PreparedTestCase<Replay>) -> Self {
        Self {
            test_case,
            events: Arc::new(Mutex::new(Vec::new())),
        }
    }

    pub fn events(&self) -> Vec<ExpectedOutputType> {
        self.events.lock().unwrap().clone()
    }
}

// MANDATORY: Config trait for environment variables
impl Config for MockProvider {
    async fn get(&self, key: &str) -> Result<String> {
        // Return mock config values - these should match .env.example
        match key {
            "BLOCK_MGT_URL" => Ok("http://localhost:8080".to_string()),
            "CC_STATIC_URL" => Ok("http://localhost:8081".to_string()),
            "AZURE_IDENTITY" => Ok("mock-identity".to_string()),
            "OUTPUT_TOPIC" => Ok("test-topic".to_string()),
            _ => Err(anyhow!("unknown config key: {key}")),
        }
    }
}

// MANDATORY: HttpRequest using Fetcher
impl HttpRequest for MockProvider {
    async fn fetch<T>(&self, request: Request<T>) -> Result<Response<Bytes>>
    where
        T: http_body::Body + Any + Send + Sync,
        T::Data: Send,
        T::Error: std::error::Error + Send + Sync + 'static,
    {
        let http_requests = self.test_case.http_requests.as_ref()
            .ok_or_else(|| anyhow!("no http_requests defined in fixture"))?;

        // Use Fetcher - do NOT manually match requests
        let fetcher = Fetcher::new(http_requests);
        fetcher.fetch(&request)
    }
}

// MANDATORY: Publisher to capture events
impl Publisher for MockProvider {
    async fn send(&self, _topic: &str, message: &Message) -> Result<()> {
        let event: ExpectedOutputType = serde_json::from_slice(&message.payload)
            .context("deserializing published event")?;
        self.events.lock().unwrap().push(event);
        Ok(())
    }
}

// MANDATORY if component uses authentication
impl Identity for MockProvider {
    async fn access_token(&self, _identity: String) -> Result<String> {
        Ok("mock_access_token".to_string())
    }
}
```

**Provider Trait Checklist**:

- [ ] `Config` - Always required (components read env vars)
- [ ] `HttpRequest` - Required if component makes HTTP calls; use `Fetcher`
- [ ] `Publisher` - Required if component publishes messages; capture events
- [ ] `Identity` - Required if ANY HTTP call uses Bearer auth

### Phase 4: Time Transformation Logic (`shift_time`)

**CRITICAL**: This function MUST be fully implemented - NOT a placeholder that returns input
unchanged.

The `shift_time` function adjusts historical fixture data so timestamps align with "now" during test
execution.

**Implementation Requirements**:

1. Accept TYPED input (e.g., `&R9kMessage`), not raw strings
2. Modify timestamp fields in the typed struct
3. Return the modified typed struct
4. Handle timezone correctly (usually Pacific/Auckland for NZ components)

**WRONG - Placeholder implementation**:

```rust
// DO NOT DO THIS - tests will fail due to timestamp validation
pub fn shift_time(input: &str, params: Option<&ReplayTransform>) -> Result<(String, Option<ReplayTransform>)> {
    Ok((input.to_string(), params.cloned()))  // WRONG: returns unchanged
}

```

**CORRECT - Full implementation**:

```rust
use chrono::{Utc, Timelike};
use chrono_tz::Pacific::Auckland;

pub fn shift_time(input: &R9kMessage, params: Option<&ReplayTransform>) -> R9kMessage {
    let delay = params.map_or(0, |p| p.delay);
    let mut request = input.clone();

    // Get first change (most fixtures have one primary change)
    let Some(change) = request.train_update.changes.get_mut(0) else {
        return request;
    };

    // 1. Set created_date to today
    let now = Utc::now().with_timezone(&Auckland);
    request.train_update.created_date = now.date_naive();

    // 2. Calculate adjusted seconds from midnight
    let from_midnight = now.num_seconds_from_midnight() as i32;
    let adjusted_secs = from_midnight - delay;

    // 3. Update the relevant time field based on event type
    if change.has_departed {
        change.actual_departure_time = adjusted_secs;
    } else if change.has_arrived {
        change.actual_arrival_time = adjusted_secs;
    }

    request
}
```

**MCP Context Required**: Use MCP to identify:

- The exact timestamp fields in the component's input type
- The timezone used in component validation logic
- Whether the component uses `seconds_from_midnight` or full `DateTime`

### Phase 5: Test Runner & Verification

Create `tests/replay.rs`.

**CRITICAL Requirements**:

1. Use `qwasr_sdk::api::Client` for handler execution - do NOT call handlers directly
2. Verify BOTH success AND error cases with assertions
3. Do NOT just return `Ok(())` for error cases - verify error code and description

#### Test Runner Implementation

```rust
use std::fs;
use qwasr_sdk::{api::Client, Error};
use augentic_test::{TestCase, TestDef};
use chrono::{Utc, Timelike};
use chrono_tz::Pacific::Auckland;

mod provider;  // tests/provider.rs
use provider::{MockProvider, Replay, shift_time};

#[tokio::test]

async fn run() {
    // CRITICAL: Use correct data path (data/replay/, not tests/data/replay/)
    for entry in fs::read_dir("data/replay").expect("should read data/replay directory") {
        let path = entry.expect("should read entry").path();
        if path.extension().map_or(false, |e| e == "json") {
            let file = fs::File::open(&path).expect("should open file");
            let test_def: TestDef<Error> = serde_json::from_reader(&file)
                .expect("should parse test definition");

            println!("Running replay: {}", path.display());
            replay(test_def).await;
        }
    }
}

async fn replay(test_def: TestDef<Error>) {
    // 1. Prepare test case with time shifting
    let test_case = TestCase::<Replay>::new(test_def).prepare(shift_time);

    // 2. Create mock provider
    let provider = MockProvider::new(test_case.clone());

    // 3. CRITICAL: Use Client abstraction (not direct handler calls)
    let client = Client::new("at").provider(provider.clone());

    // 4. Execute handler via client
    let result = client.request(test_case.input.expect("replay test input expected")).await;

    // 5. Get captured events from provider
    let actual_events = provider.events();

    // 6. Verify based on expected output
    let expected_result = test_case.output.expect("replay test output expected");

    match expected_result {
        Ok(expected_events) => {
            // SUCCESS CASE: Verify events match
            assert_eq!(
                expected_events.len(),
                actual_events.len(),
                "event count mismatch: expected {}, got {}",
                expected_events.len(),
                actual_events.len()
            );

            for (expected, mut actual) in expected_events.into_iter().zip(actual_events) {
                // Check timestamp drift (allow small window for execution time)
                let now = Utc::now().with_timezone(&Auckland);
                let diff = now.timestamp() - actual.message_data.timestamp.timestamp();
                assert!(diff.abs() < 3, "timestamp drift too high: {diff} seconds");

                // Normalize timestamps for equality comparison
                actual.received_at = expected.received_at;
                actual.message_data.timestamp = expected.message_data.timestamp;

                // Deep equality check via JSON
                let json_expected = serde_json::to_value(&expected).unwrap();
                let json_actual = serde_json::to_value(&actual).unwrap();
                assert_eq!(json_expected, json_actual, "event mismatch");
            }
        }
        Err(expected_error) => {
            // ERROR CASE: MUST verify error - do NOT just return Ok(())
            let actual_error = result.expect_err("expected handler to return error");

            assert_eq!(
                actual_error.code(),
                expected_error.code(),
                "error code mismatch: expected '{}', got '{}'",
                expected_error.code(),
                actual_error.code()
            );

            assert_eq!(
                actual_error.description(),
                expected_error.description(),
                "error description mismatch"
            );
        }
    }
}
```

**WRONG - Error not verified**:

```rust
// DO NOT DO THIS - errors silently pass
Some(Err(expected_error)) => {
    debug!(code = %expected_error.code, "expecting handler error");
    Ok(())  // WRONG: test passes even if wrong error returned
}
```

## Versioning the Prompt (staged migration alignment)

- **Component type tiers**: Maintain minor versions per component family (e.g., `v1-http-only`,
`v1.1-http-messaging`, `v1.2-sql`).
- **Complexity tiers**: Add suffixes for time-shifted vs non-time-sensitive components (`-timeful`,
`-timeless`).
- **Changelog discipline**: When expanding required providers or schema expectations, bump the minor
version and document breaking expectations for fixtures.

## Performance and Security Notes

- **Async efficiency**: Prefer reusing `Fetcher` instances per test_case; avoid cloning large
fixtures repeatedly. Use `Arc<Mutex<...>>` only where mutation is required and keep lock scope
short.
- **Tracing**: Use `tracing` for structured logs; avoid logging sensitive payloads. Log only
summaries/hashes when needed.

## Glossary

- **qwasr_sdk**: Guest Rust SDK exposing provider traits (HTTP, messaging, config, identity,
telemetry) for Augentic WASM components.
- **Replay/Replayer**: Test harness that replays recorded inputs/fixtures against a WASM component.
- **Fixture**: `augentic_test::Fixture` implementation describing how to map JSON test data to
component inputs/outputs.
- **Fetcher**: Helper used to match and mock HTTP requests from fixtures.

## JSON Data Schema (EXACT FORMAT - DO NOT DEVIATE)

The test files in `data/replay/` follow this EXACT structure. Your deserialization must match.

### Success Case Example

```json
{
    "input": "<CCO xmlns:xsi=\"...\">...</CCO>",
    "params": { "delay": 120 },
    "http_requests": [
        {
            "path": "/api/blocks",
            "method": "GET",
            "response": {
                "status": 200,
                "body": ["vehicle1", "vehicle2"]
            }
        }
    ],
    "output": {
        "success": [
            {
                "receivedAt": "2025-01-30T10:00:00Z",
                "eventType": "Location",
                "messageData": {
                    "timestamp": "2025-01-30T10:00:00.000000000Z"
                },
                "remoteData": {
                    "externalId": "5271",
                    "remoteName": "vehicle1"
                },
                "locationData": {
                    "latitude": -36.84448,
                    "longitude": 174.76915
                }
            }
        ]
    }
}
```

### Error Case Example

```json
{
    "input": "<CCO>...</CCO>",
    "params": { "delay": 506 },
    "output": {
        "failure": {
            "BadRequest": {
                "code": "bad_time",
                "description": "outdated by 506 seconds"
            }
        }
    }
}
```

### Key Schema Points

1. **output.success** (lowercase) contains array of events
2. **output.failure** (lowercase) contains error wrapper
3. Error wrapper has variant name (`BadRequest`) with nested `{ code, description }`
4. **params.delay** is integer seconds (positive = message is old)
5. **http_requests** is optional array of request/response mocks
6. **input** is typically a raw XML string that needs parsing

## Task

Generate the full code for:

1. `tests/provider.rs`
2. `tests/replay.rs`

Ensure all imports are resolved and the code compiles with `augentic-test`, `qwasr-sdk`, and
`chrono`.

ALSO OUTPUT A TEST CONSTRUCTION REPORT DOC ON WHAT ACTIONS WHERE TAKEN AND THE STATE OF THE TEST AND
REPLAYER

## POST-GENERATION VERIFICATION CHECKLIST (MANDATORY)

Before submitting, verify ALL of the following:

**Fixture Implementation**:

- [ ] Implements `augentic_test::Fixture` trait (not custom `TestFixture`)
- [ ] All 4 associated types defined: `Error`, `Input`, `Output`, `TransformParams`
- [ ] All 4 trait methods implemented: `from_data`, `input`, `params`, `output`
- [ ] `from_data` handles XML string -> typed struct conversion if needed

**MockProvider**:

- [ ] Implements ALL provider traits used by component (check handler bounds)
- [ ] `Identity` included if ANY HTTP call uses Bearer auth
- [ ] Uses `Fetcher` for HTTP mocking (not manual request matching)
- [ ] Captures events in `Arc<Mutex<Vec<_>>>` for post-execution access

**shift_time**:

- [ ] Accepts TYPED input (not raw string)
- [ ] Actually modifies timestamps (not a pass-through placeholder)
- [ ] Uses correct timezone (check component's validation logic)
- [ ] Updates `created_date` to today
- [ ] Updates `actual_arrival_time` or `actual_departure_time` based on flags

**Test Runner**:

- [ ] Uses `Client::new("owner").provider(...)` abstraction
- [ ] Does NOT call handlers directly
- [ ] Data path is `data/replay/` (not `tests/data/replay/`)

**Error Verification**:

- [ ] Success cases verify event count AND content
- [ ] Error cases call `expect_err()` and assert code/description
- [ ] NO `Ok(())` returns for error cases without verification

**Imports**:

- [ ] `augentic_test` in dev-dependencies
- [ ] All imports resolve: `Fetcher`, `Fixture`, `PreparedTestCase`, `TestCase`, `TestDef`
- [ ] `qwasr_sdk::api::Client` imported for test execution
