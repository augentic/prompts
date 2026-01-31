---
name: migrate-code
description: Orchestrate full TypeScript to Rust WASM migration pipeline by coordinating clone, analyse, generate, and test skills.
argument-hint: [repo-url] [workspace] [crate-name] OR [config.toml]
---

# Migrate Code Skill

## Overview

Coordinate a complete migration from a TypeScript repository to a Rust WASM crate by invoking the individual skills in sequence. This orchestrator handles the full workflow from cloning to test generation.

Supports two invocation modes:

- **Positional arguments**: Quick single migrations
- **Config file**: Reproducible single or batch migrations

## Required Permissions

⚠️ **CRITICAL**: This skill runs the full migration pipeline including git clone, file I/O, and cargo builds.

**FIRST ACTION**: Execute a simple command with `required_permissions: ["all"]` to establish unrestricted context:

```bash
# Request all permissions upfront (first command only)
echo "Initializing migration pipeline with full permissions"
```

After this initial permission grant, all subsequent commands inherit the permission context.

---

## Arguments

### Mode 1: Positional Arguments

```agent
migrate-code [repo-url] [workspace] [crate-name]
```

1. **Repository URL** (`$ARGUMENTS[0]`): Git repository URL containing TypeScript source
2. **Workspace** (`$ARGUMENTS[1]`): Base directory for all operations (e.g., `./migrations`)
3. **Crate Name** (`$ARGUMENTS[2]`): Name for the output Rust crate (e.g., `my_component`)

### Mode 2: Config File

```agent
migrate-code [config.toml]
migrate-code              # Auto-detects ./migrate.toml
```

1. **Config Path** (`$ARGUMENTS[0]`): Path to TOML config file (optional if `migrate.toml` exists in cwd)

### Detection Logic

```agent
IF $ARGUMENTS[0] ends with ".toml":
    Parse as config file
ELSE IF no arguments AND ./migrate.toml exists:
    Parse ./migrate.toml
ELSE:
    Use positional arguments (require all 3)
```

---

## Config File Format

### Single Migration

```toml
# migrate.toml
repo = "https://github.com/org/my-component"
workspace = "./migrations"
crate = "my_component"
```

### Batch Migration

```toml
# migrate.toml
workspace = "./migrations"

[[projects]]
repo = "https://github.com/org/component-a"
crate = "component_a"

[[projects]]
repo = "https://github.com/org/component-b"
crate = "component_b"

[[projects]]
repo = "https://github.com/org/component-c"
crate = "component_c"
```

### Config Fields

| Field | Required | Description |
| ----- | -------- | ----------- |
| `repo` | Yes | Git repository URL |
| `workspace` | Yes | Base directory for outputs |
| `crate` | Yes | Output crate name |
| `skip` | No | Set `true` to skip this project in batch |

---

## Derived Paths

```agent
$TYPESCRIPT = $ARGUMENTS[1]/legacy/<repo_name>      # Cloned TypeScript repo
$IR_PATH    = $ARGUMENTS[1]/ir/$ARGUMENTS[2].ir.md  # Intermediate representation
$RUST_CRATE = $ARGUMENTS[1]/crates/$ARGUMENTS[2]    # Generated Rust crate
```

---

## Pipeline Stages

### Stage 1: Clone Repository

**Skill**: `clone-repo`

**Action**: Clone the TypeScript repository to the workspace.

```agent
Input:  $ARGUMENTS[0] $ARGUMENTS[1]/legacy
Output: $TYPESCRIPT (cloned repository path)
```

**Validation**:

- [ ] Repository cloned successfully
- [ ] Contains TypeScript/JavaScript files
- [ ] `package.json` exists

### Stage 2: Analyse Codebase

**Skill**: `analyse-codebase`

**Action**: Generate IR from the TypeScript source.

```agent
Input:  $TYPESCRIPT
Output: $IR_PATH (intermediate representation file)
```

**Validation**:

- [ ] IR file created at `$IR_PATH`
- [ ] IR contains Component, Structures, Business Logic sections
- [ ] External API surfaces documented

### Stage 3: Generate Crate

**Skill**: `generate-crate`

**Action**: Generate Rust crate from IR following QWASR SDK patterns.

```agent
Input:  $IR_PATH, $TYPESCRIPT (reference)
Output: $RUST_CRATE (complete Rust crate)
```

**Validation**:

- [ ] `Cargo.toml` exists with correct dependencies
- [ ] `src/lib.rs` and handler files exist
- [ ] `Migration.md`, `Architecture.md`, `.env.example` exist
- [ ] `cargo check` passes

### Stage 4: Generate Tests

**Skill**: `generate-tests`

**Action**: Generate Replayer test harness for the component.

```agent
Input:  $RUST_CRATE
Output: $RUST_CRATE/tests/provider.rs, $RUST_CRATE/tests/replay.rs
```

**Validation**:

- [ ] Test files created
- [ ] `cargo test --no-run` compiles successfully

### Stage 5: Final Verification

**Action**: Run comprehensive checks on the generated crate.

```bash
cd $RUST_CRATE
cargo fmt --check
cargo clippy -- -D warnings
cargo test
```

---

## Execution Flow

```text
┌─────────────────────────────────────────────────────────────┐
│                       Code Migration Pipeline                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. clone-repo                                              │
│     └─► $TYPESCRIPT (cloned TypeScript source)              │
│                          │                                  │
│                          ▼                                  │
│  2. analyse-codebase                                        │
│     └─► $IR_PATH (intermediate representation)              │
│                          │                                  │
│                          ▼                                  │
│  3. generate-crate                                          │
│     └─► $RUST_CRATE (Rust WASM component)                   │
│                          │                                  │
│                          ▼                                  │
│  4. generate-tests                                          │
│     └─► tests/provider.rs, tests/replay.rs                  │
│                          │                                  │
│                          ▼                                  │
│  5. Verify & Report                                         │
│     └─► Migration complete                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Process

### Step 0a: Request Permissions

**FIRST**: Execute this command to establish unrestricted permission context:

```bash
echo "Initializing migration pipeline with full permissions" && date
```

**REQUIRED**: Use `required_permissions: ["all"]` for this initial command only. All subsequent commands will inherit this permission level.

### Step 0b: Resolve Configuration

Determine invocation mode and extract project(s):

```agent
IF $ARGUMENTS[0] ends with ".toml":
    config_path = $ARGUMENTS[0]
ELSE IF no arguments:
    config_path = "./migrate.toml" (must exist)
ELSE:
    # Positional mode - single project
    projects = [{ repo: $ARGUMENTS[0], workspace: $ARGUMENTS[1], crate: $ARGUMENTS[2] }]
    GOTO Step 1

# Config mode - parse TOML
config = parse_toml(config_path)

IF config has [[projects]] array:
    # Batch mode
    workspace = config.workspace
    projects = config.projects.filter(p => !p.skip)
ELSE:
    # Single project in config
    projects = [{ repo: config.repo, workspace: config.workspace, crate: config.crate }]
```

### Step 1: Setup Workspace

For each project, create directory structure:

```bash
mkdir -p $workspace/legacy
mkdir -p $workspace/ir
mkdir -p $workspace/crates
```

### Step 2: Execute Pipeline

For each project (sequentially):

1. Announce project start: `"Migrating: {repo} → {crate}"`
2. Set derived paths for this project
3. Execute all 5 stages
4. Report project success/failure
5. Continue to next project (failures don't block other projects in batch mode)

### Step 3: Report Results

After completion, provide summary:

```text
Migration Complete
==================

Source:     $ARGUMENTS[0]
Crate:      $RUST_CRATE
IR:         $IR_PATH

Files Generated:
  - src/lib.rs
  - src/error.rs
  - src/types.rs
  - src/handlers.rs
  - Cargo.toml
  - Migration.md
  - Architecture.md
  - .env.example
  - tests/provider.rs
  - tests/replay.rs

Verification:
  ✓ cargo check passed
  ✓ cargo clippy passed
  ✓ cargo test passed

Next Steps:
  1. Review Migration.md for transformation decisions
  2. Review TODO markers in generated code
  3. Add test fixtures to data/replay/
  4. Run full test suite
```

---

## Error Handling

### Stage Failure

If any stage fails:

1. Report the failure with details
2. Preserve partial outputs for debugging
3. Suggest remediation steps
4. Do not proceed to subsequent stages

### Common Issues

| Issue | Stage | Resolution |
| ----- | ----- | ---------- |
| Clone fails | 1 | Check URL, auth, network |
| IR incomplete | 2 | Review TypeScript structure |
| Cargo check fails | 3 | Fix compilation errors |
| Tests don't compile | 4 | Check type mismatches |

---

## Batch Migration

Use a config file with `[[projects]]` array:

```toml
# migrate.toml
workspace = "./migrations"

[[projects]]
repo = "https://github.com/org/repo1"
crate = "repo1"

[[projects]]
repo = "https://github.com/org/repo2"
crate = "repo2"

[[projects]]
repo = "https://github.com/org/repo3"
crate = "repo3"
skip = true  # Temporarily skip this one
```

Then run:

```agent
migrate-code migrate.toml
```

### Batch Behavior

- Projects are processed sequentially
- Each project's success/failure is reported independently
- A failed project does not block subsequent projects
- Final summary shows all results

### Batch Summary Output

```text
Batch Migration Complete
========================

✓ repo1 → component_a (success)
✓ repo2 → component_b (success)
✗ repo3 → component_c (failed at Stage 3: cargo check)
⊘ repo4 → component_d (skipped)

Passed: 2/4
Failed: 1/4
Skipped: 1/4
```

---

## Examples

### Quick Single Migration

```agent
migrate-code https://github.com/org/my-service ./migrations my_service
```

### Using Config File

```agent
migrate-code ./projects/migrate.toml
```

### Auto-Detect Config

```agent
# With ./migrate.toml present in cwd:
migrate-code
```

---

## Important Notes

- Each stage builds on the previous stage's output
- Validation gates prevent cascading failures
- All intermediate artifacts are preserved for debugging
- The pipeline can be resumed from any stage if variables are set correctly
- Generated crates target `wasm32-wasip1` or `wasm32-wasip2`
- Config file mode is recommended for reproducibility and batch migrations
