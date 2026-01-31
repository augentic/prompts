# Rust Migration Plugin

A comprehensive plugin for migrating TypeScript projects to Rust WASM crates targeting the QWASR SDK and wasmtime runtime.

**This plugin is the single source of truth for all migration skills.**

## Installation

The skills in this plugin are automatically available to Cursor agents via the `<agent_skills>` section in the system prompt.

## Skills

### Core Skills

| Skill | Arguments | Description |
| ------- | ----------- |------------- |
| `clone-repo` | `[repo-url] [target-dir]` | Clone a git repository with validation |
| `analyse-codebase` | `[typescript-source] [ir-output-path]` | Generate IR from TypeScript source |
| `generate-crate` | `[ir-path] [typescript-source] [rust-crate-output]` | Generate Rust crate from IR |
| `generate-tests` | `[rust-crate-path]` | Generate Replayer test harness |

### Orchestration

| Skill | Arguments | Description |
| ------ | --------- | ----------- |
| `migrate-code` | `[repo-url] [workspace] [crate-name]` | Full migration pipeline |
| `migrate-code` | `[config.toml]` | Config-driven migration (single or batch) |

### Reference

| Skill | Description |
| ----- | ----------- |
| `rust-migration` | General Rust migration guide (type mapping, error handling, async patterns) |

## Quick Start

### Single Migration (Positional Arguments)

```agent
migrate-code https://github.com/org/my-service ./migrations my_service
```

### Single Migration (Config File)

```toml
# migrate.toml
repo = "https://github.com/org/my-service"
workspace = "./migrations"
crate = "my_service"
```

```agent
migrate-code migrate.toml
```

### Batch Migration

```toml
# migrate.toml
workspace = "./migrations"

[[projects]]
repo = "https://github.com/org/service-a"
crate = "service_a"

[[projects]]
repo = "https://github.com/org/service-b"
crate = "service_b"

[[projects]]
repo = "https://github.com/org/service-c"
crate = "service_c"
skip = true  # Temporarily skip
```

```
migrate-code migrate.toml
```

### Individual Skills

```agent
clone-repo https://github.com/org/repo ./source
analyse-codebase ./source/repo ./ir/component.ir.md
generate-crate ./ir/component.ir.md ./source/repo ./crates/my_crate
generate-tests ./crates/my_crate
```

## Pipeline Workflow

```text
1. clone-repo        → TypeScript source
        ↓
2. analyse-codebase  → Intermediate Representation (IR)
        ↓
3. generate-crate    → Rust WASM component
        ↓
4. generate-tests    → Replayer test harness
```

## Key Features

- **QWASR SDK Integration**: Generated code follows provider-based dependency injection patterns
- **Stateless WASM Components**: All generated code is stateless and WASM-compatible
- **Replayer Testing**: Test harness for running components against recorded production data
- **Config-Driven Migrations**: TOML config for reproducible single or batch migrations
- **Hybrid MCP Strategy**: Essential patterns embedded, with MCP for latest updates

## Requirements

- Git (for clone-repo)
- Rust toolchain with `wasm32-wasip1` or `wasm32-wasip2` target
- Access to `qwasr-sdk` and `augentic-test` crates

## Permissions

This plugin is designed to request `["all"]` permissions at the start of skill execution to provide a seamless experience.

### How It Works

1. **First Command**: Each skill executes a simple initialization command with `required_permissions: ["all"]`
2. **Single Approval**: You approve permissions once at the beginning
3. **Seamless Execution**: All subsequent commands inherit the permission context without additional prompts

### Why "all" Permissions?

The migration pipeline requires:
- **Network**: Cloning repositories from remote Git services (GitHub, Bitbucket, GitLab)
- **Git Write**: Creating and managing local git repositories
- **File I/O**: Reading TypeScript source, generating Rust crates, creating test files
- **Cargo**: Running `cargo check`, `cargo test`, `cargo fmt`, `cargo clippy`

Rather than prompting multiple times throughout execution, skills request all permissions upfront for a better user experience.
