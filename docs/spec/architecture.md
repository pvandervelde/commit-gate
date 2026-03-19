# Architecture

## Business Logic

The core domain is the check pipeline and its execution model. This is pure logic with no external dependencies.

### Domain Operations

- **Load and validate configuration** from TOML into typed internal structures
- **Build a check pipeline** from configuration (ordered sequence of checks)
- **Execute the pipeline** against a set of staged files, collecting results
- **Evaluate path restrictions** against a mode's allowed paths
- **Scan file content for patterns** (stub detection)
- **Validate commit messages** against format rules
- **Format gate results** for human or machine consumption

### Domain Types

- Configuration (parsed and validated `.commitgate.toml`)
- CheckDefinition (a single check's configuration: name, type, parameters)
- Mode (named path restriction set)
- CheckResult (pass/fail + diagnostics for a single check)
- GateResult (aggregate outcome of the full pipeline)
- FileSource (abstraction over how files and their content are obtained — either from the git index via `--staged`, or from the filesystem via `--files`)
- StagedFile (path + content of a file; the content source depends on the FileSource)
- StubPattern (a pattern to scan for, with file extension filter)
- Remediation (actionable instruction for fixing a failure)

### Business Rules

- Checks execute in the order defined in configuration, never reordered
- In fail-fast mode, pipeline halts on first check failure
- Path check always runs first when a mode is active
- A check that times out is treated as a failure, not an error
- When `--staged` is used, file content is read from the git index (`git show :path`), not the working tree
- When `--files` is used, file content is read from the filesystem; the git staging area is not consulted for content
- `--staged` and `--files` are mutually exclusive; providing both is a gate error
- Path check (mode restrictions) applies equally to `--staged` and `--files` file lists
- Output truncation is applied per-check, not globally
- Exit code semantics: 0 = all checks passed, 1 = at least one check failed, 2 = gate error (bad config, missing tool, etc.)

## External System Interfaces

### Git

- Read list of staged files (`--staged` mode): `git diff --cached --name-only --diff-filter=ACMR`
- Read staged file content (`--staged` mode): `git show :<path>`
- Determine repository root: `git rev-parse --show-toplevel`
- `commitgate check` never writes to git. `commitgate mcp` writes to git as part of `propose_commit` (see security.md).

### Filesystem

- Read `.commitgate.toml` configuration file
- Read file content (`--files` mode): read each specified path directly from the filesystem
- Read `COMMITGATE_MODE` environment variable

### Subprocess Execution

- Run command checks as child processes with captured stdout/stderr
- Apply per-process timeouts
- Template `{files}` substitution in command strings

### Standard I/O

- Write human or JSON output to stdout
- Write unexpected errors to stderr
- Read nothing from stdin (the gate never blocks for input)

## Infrastructure Implementations

These are the concrete implementations of the external system interfaces:

### GitCli

Implements git operations by shelling out to the `git` binary. This is the only implementation needed — there is no in-process libgit2 dependency. The git binary is assumed to be in PATH.

### ProcessRunner

Implements subprocess execution using `std::process::Command`. Handles timeout enforcement via spawned threads or async tasks, stdout/stderr capture, and exit code interpretation.

### TomlParser

Implements configuration loading using the `toml` crate. Handles file discovery (walk up from cwd), parsing, validation, and default merging.

## Dependency Flow

```
CLI (entry point)
 ├── ConfigLoader
 │    └── TomlParser (infrastructure)
 ├── Pipeline
 │    ├── FileSource (abstraction)
 │    │    ├── StagedFiles    (infrastructure — used with --staged / hook)
 │    │    └── FilesystemFiles (infrastructure — used with --files)
 │    ├── PathCheck (built-in, pure logic)
 │    ├── StubCheck (built-in, pure logic + FileSource)
 │    ├── MessageCheck (built-in, pure logic)
 │    └── CommandCheck
 │         └── ProcessRunner (infrastructure)
 └── OutputFormatter (pure logic)

MCP Server (entry point, alternative to CLI)
 ├── ConfigLoader         (shared)
 ├── Pipeline             (shared)
 ├── GitCli               (shared — used to stage + commit on success)
 └── McpTransport         (stdio MCP protocol framing)
```

Business logic (Pipeline, checks, OutputFormatter) depends only on abstractions. Infrastructure (GitCli, ProcessRunner, TomlParser) implements those abstractions. Both the CLI and MCP server are composition roots that wire the same core together.

## Module Structure (Rust)

```
commitgate/
├── Cargo.toml
└── src/
    ├── main.rs              # CLI + MCP entry point, argument parsing, wiring
    ├── config.rs            # Configuration types, TOML parsing, validation
    ├── pipeline.rs          # Pipeline execution, check orchestration
    ├── checks/
    │   ├── mod.rs           # Check trait, CheckResult type
    │   ├── path.rs          # PathCheck implementation
    │   ├── stubs.rs         # StubCheck implementation
    │   ├── message.rs       # MessageCheck implementation
    │   └── command.rs       # CommandCheck implementation
    ├── files.rs             # FileSource trait + StagedFiles (git index) and
    │                        # FilesystemFiles (--files) implementations
    ├── git.rs               # Git operations (staged files, content, commit)
    ├── output.rs            # Human and JSON output formatting
    ├── process.rs           # Subprocess execution with timeout
    └── mcp.rs               # MCP stdio server: tool registration, propose_commit handler
```

This is a single crate. The domain is small enough that separate crates add overhead without benefit. Module boundaries enforce separation. The MCP server (`mcp.rs`) reuses the same `Pipeline`, `ConfigLoader`, and `GitCli` as the CLI — it adds only the protocol transport layer.
