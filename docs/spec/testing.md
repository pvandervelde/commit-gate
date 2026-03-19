# Testing Strategy

## Test Categories

### Unit Tests
Test individual components in isolation with mock dependencies.

**ConfigLoader:** Parse valid/invalid TOML, verify defaults, validate error messages. No git or filesystem access — test against in-memory TOML strings.

**PathCheck:** Given a mode and file list, verify accept/reject decisions. Pure logic, no I/O.

**StubCheck:** Given file content and patterns, verify detection and line reporting. Pure logic, no I/O.

**MessageCheck:** Given a message string and rules, verify pass/fail decisions. Pure logic, no I/O.

**OutputFormatter:** Given a GateResult, verify human and JSON output strings. Pure logic, no I/O.

**Pipeline:** Given a mock set of checks, verify execution order, fail-fast behaviour, and result aggregation. Checks are mocked — the pipeline tests orchestration, not individual check logic.

### Integration Tests
Test the full pipeline against a real (temporary) git repository.

**Lifecycle tests:**
1. Create a temp directory, `git init`
2. Write a `.commitgate.toml`
3. Create and stage files
4. Run `commitgate check --staged`
5. Verify exit code and output

These tests cover the real git integration, config discovery, and end-to-end pipeline execution.

**Scenarios to cover:**
- All checks pass → exit 0
- Formatting fails → exit 1, correct diagnostic
- Stub detected → exit 1, file and line reported
- Path violation → exit 1, file and mode reported
- Missing tool → exit 2, tool name in error
- No config file → exit 2, clear error
- Empty staging area → exit 0, nothing to check
- Timeout → exit 1, timeout diagnostic
- JSON output parseable
- Non-TTY auto-selects JSON

### Contract Tests
Test that the external system interfaces (Git, Process) behave as expected.

**GitCli contract:**
- `staged_files()` returns correct paths after `git add`
- `staged_content(path)` returns staged content, not working tree content
- Partial staging (`git add -p`) returns only the staged portion
- Deleted files are excluded

**ProcessRunner contract:**
- Exit code 0 → pass
- Exit code non-zero → fail with captured output
- Timeout kills the process
- Output is captured from both stdout and stderr

**McpServer contract:**
- `initialize` handshake succeeds over stdio
- `tools/list` response includes `propose_commit` with correct input schema
- `propose_commit` with passing checks executes git commit and returns `success: true`
- `propose_commit` with failing checks returns `success: false` and does NOT execute git commit
- Gate errors are returned as MCP protocol errors, not as `GateResult`

### Property Tests
Test invariants that should hold across many inputs.

- For any valid config, the pipeline produces a GateResult with the correct number of check results (one per executed check)
- For any file path and any mode, the path check's decision is deterministic
- For any stub pattern and any file content, the stub check either finds it at the correct line or doesn't find it — never a wrong line
- JSON output is always valid JSON (parseable by a standard parser)

## Test Doubles

### MockGit
Returns predefined staged file lists and content. Used in unit tests for Pipeline, StubCheck, and PathCheck.

### MockProcess
Returns predefined exit codes and output. Used in unit tests for CommandCheck.

### MockCheck
Implements the Check trait with predefined pass/fail results. Used in Pipeline orchestration tests.

## Test Environment

Integration tests require:
- `git` in PATH (any version ≥ 2.20)
- A temporary directory for each test (cleaned up automatically)
- `cargo fmt` and `cargo clippy` available (for Rust-specific command check tests)

Tests that require external tools should be gated behind a feature flag or cargo test filter so the core tests can run without a full Rust toolchain.

## Coverage Targets

- Unit tests: 90%+ line coverage on business logic (config, pipeline, checks, output)
- Integration tests: one test per assertion in `assertions.md`
- Every edge case in `edge-cases.md` has a corresponding test
- Every check failure produces a non-empty `remediation` field
