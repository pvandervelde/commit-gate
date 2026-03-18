# Behavioral Assertions

These assertions are testable specifications. Each maps to one or more tests. They define what must be true about the system, derived from the requirements and architectural decisions.

## Configuration

### CG-CFG-01: Config discovered by walking up from cwd

- Given: `.commitgate.toml` exists at `/home/dev/project/.commitgate.toml`
- And: the current working directory is `/home/dev/project/src/auth/`
- When: the gate loads configuration
- Then: it discovers and loads `/home/dev/project/.commitgate.toml`

### CG-CFG-02: Missing config produces clear error

- Given: no `.commitgate.toml` exists in any parent directory
- When: the gate attempts to load configuration
- Then: it exits with code 2
- And: the error message names the file it was looking for
- And: the error message is human-readable, not a Rust panic

### CG-CFG-03: Malformed TOML produces specific diagnostic

- Given: `.commitgate.toml` contains invalid TOML syntax
- When: the gate attempts to load configuration
- Then: it exits with code 2
- And: the error message includes the line number and nature of the syntax error

### CG-CFG-04: Unknown check type produces clear error

- Given: `.commitgate.toml` defines a check with `type = "unknown"`
- When: the gate validates configuration
- Then: it exits with code 2
- And: the error names the offending check and the invalid type value

### CG-CFG-05: Default values applied for optional fields

- Given: `.commitgate.toml` omits `fail_fast` and `max_output_lines`
- When: the gate loads configuration
- Then: `fail_fast` defaults to `true`
- And: `max_output_lines` defaults to 30

---

## Pipeline Execution

### CG-PIPE-01: Checks run in configured order

- Given: checks are configured as `["formatting", "no-stubs", "tests"]`
- When: the pipeline executes
- Then: formatting runs before no-stubs, and no-stubs runs before tests
- And: the order is reflected in the gate result's check list

### CG-PIPE-02: Fail-fast halts on first failure

- Given: `fail_fast = true` (default)
- And: the formatting check fails
- When: the pipeline executes
- Then: no-stubs and tests do NOT execute
- And: the gate result contains exactly one check result (formatting, failed)

### CG-PIPE-03: Run-all mode reports all failures

- Given: `fail_fast = false`
- And: formatting fails and no-stubs fails
- When: the pipeline executes
- Then: all three checks execute
- And: the gate result contains three check results
- And: both formatting and no-stubs are marked as failed

### CG-PIPE-04: All checks pass produces success result

- Given: all configured checks pass
- When: the pipeline executes
- Then: the gate result has `success: true`
- And: every check result has `passed: true`

### CG-PIPE-05: Empty staged file list skips all checks

- Given: no files are staged
- When: the pipeline executes
- Then: the gate result has `success: true`
- And: no checks are executed
- And: a message indicates no staged files were found

---

## Path Check

### CG-PATH-01: Files within writable paths are accepted

- Given: mode `tester` with `writable = ["tests"]`
- And: staged files are `tests/auth_test.rs` and `tests/session_test.rs`
- When: the path check runs
- Then: the check passes

### CG-PATH-02: Files outside writable paths are rejected

- Given: mode `tester` with `writable = ["tests"]`
- And: staged files include `src/auth.rs`
- When: the path check runs
- Then: the check fails
- And: the diagnostic lists `src/auth.rs` as the violating file
- And: the remediation names the allowed paths

### CG-PATH-03: No mode set skips path check

- Given: `COMMITGATE_MODE` is not set
- When: the pipeline executes
- Then: the path check does not run (even if defined in the sequence)

### CG-PATH-04: Unknown mode produces clear error

- Given: `COMMITGATE_MODE=reviewer` but no `[modes.reviewer]` exists in config
- When: the gate starts
- Then: it exits with code 2
- And: the error names the unknown mode and lists available modes

### CG-PATH-05: Deeply nested paths are matched correctly

- Given: mode `coder` with `writable = ["src"]`
- And: staged file is `src/auth/service/validation.rs`
- When: the path check runs
- Then: the check passes (file is under `src/`)

### CG-PATH-06: `always_writable` paths are accepted in all modes

- Given: mode `tester` with `writable = ["tests"]`
- And: `[modes._global] always_writable = ["docs", ".llm"]`
- And: staged files are `tests/auth_test.rs` and `docs/README.md`
- When: the path check runs
- Then: the check passes (both files are within allowed paths)

### CG-PATH-07: Default `always_writable` applies when `[modes._global]` is absent

- Given: a mode is active
- And: no `[modes._global]` section is defined in config
- And: staged file is `docs/README.md`
- When: the path check runs
- Then: `docs/README.md` is accepted (default `always_writable = ["docs", ".llm"]` applies)

---

## Stub Check

### CG-STUB-01: Stub patterns are detected in staged content

- Given: `patterns = ["todo!()"]` and `extensions = [".rs"]`
- And: staged file `src/auth.rs` contains `todo!()` on line 42
- When: the stub check runs
- Then: the check fails
- And: the diagnostic reports `src/auth.rs:42: todo!()`

### CG-STUB-02: Patterns not present in staged files pass

- Given: `patterns = ["todo!()"]`
- And: no staged file contains `todo!()`
- When: the stub check runs
- Then: the check passes

### CG-STUB-03: Only configured extensions are scanned

- Given: `extensions = [".rs"]`
- And: staged file `README.md` contains `// TODO`
- When: the stub check runs
- Then: `README.md` is not scanned (wrong extension)
- And: the check passes (no matches in `.rs` files)

### CG-STUB-04: Staged content is checked, not working tree

- Given: staged content of `src/auth.rs` does NOT contain `todo!()`
- And: working tree content of `src/auth.rs` DOES contain `todo!()`
- When: the stub check runs
- Then: the check passes (only staged content matters)

### CG-STUB-05: Multiple patterns and multiple files are all reported

- Given: `patterns = ["todo!()", "unimplemented!()"]`
- And: `src/auth.rs` contains `todo!()` on line 10
- And: `src/session.rs` contains `unimplemented!()` on line 25
- When: the stub check runs
- Then: the check fails
- And: both matches are reported in the diagnostic

---

## Command Check

### CG-CMD-01: Exit code 0 is a pass

- Given: a command check configured as `command = "cargo fmt --check"`
- And: `cargo fmt --check` exits with code 0
- When: the command check runs
- Then: the check passes

### CG-CMD-02: Non-zero exit code is a failure

- Given: a command check configured as `command = "cargo fmt --check"`
- And: `cargo fmt --check` exits with code 1
- When: the command check runs
- Then: the check fails
- And: the captured stdout/stderr is included in the diagnostic

### CG-CMD-03: {files} template is substituted with staged file list

- Given: a command check configured as `command = "prettier --check {files}"`
- And: staged files are `src/app.ts` and `src/util.ts`
- When: the command check runs
- Then: the executed command is `prettier --check src/app.ts src/util.ts`

### CG-CMD-04: Check timeout kills the process

- Given: a command check with `timeout = 5`
- And: the command hangs for longer than 5 seconds
- When: the command check runs
- Then: the process is killed after 5 seconds
- And: the check fails with a timeout diagnostic
- And: the remediation indicates the timeout duration

### CG-CMD-05: Output is truncated to configured limit

- Given: `max_output_lines = 10`
- And: the command produces 500 lines of output
- When: the command check runs and fails
- Then: the diagnostic contains at most 10 lines of output
- And: a truncation notice indicates how many lines were omitted

### CG-CMD-06: Missing command produces gate error

- Given: a command check configured as `command = "nonexistent-tool --check"`
- And: `nonexistent-tool` is not in PATH
- When: the command check runs
- Then: the gate exits with code 2 (gate error, not check failure)
- And: the error identifies the missing command

---

## Message Check

### CG-MSG-01: Message meeting all rules passes

- Given: `min_length = 15` and `format = "conventional"`
- And: commit message is `feat(auth): Add login rate limiting`
- When: the message check runs
- Then: the check passes

### CG-MSG-02: Short message is rejected

- Given: `min_length = 15`
- And: commit message is `fix`
- When: the message check runs
- Then: the check fails
- And: the diagnostic states the minimum length requirement

### CG-MSG-03: Non-conventional format is rejected

- Given: `format = "conventional"`
- And: commit message is `Added some stuff to auth`
- When: the message check runs
- Then: the check fails
- And: the diagnostic explains conventional commit format

### CG-MSG-04: Forbidden patterns are rejected

- Given: `forbidden_patterns = ["WIP", "fixup", "squash"]`
- And: commit message is `WIP: auth changes`
- When: the message check runs
- Then: the check fails
- And: the diagnostic identifies the matched forbidden pattern

### CG-MSG-05: Message check skipped when no message is available (pre-commit context)

- Given: `message` check is in `sequence`
- And: no `--message` flag is provided (typical pre-commit hook invocation)
- When: the pipeline reaches the message check
- Then: the message check is skipped without failure
- And: a warning is written to stderr explaining that the message cannot be validated in the pre-commit hook context
- And: the warning recommends installing the gate as a commit-msg hook to validate the final message
- Note: this is not a gate error (code 2) — it is an expected condition when the user has not provided a message

---

## Output

### CG-OUT-01: JSON output is a single valid JSON object

- Given: `--json` flag is set
- When: the gate produces output
- Then: stdout contains exactly one JSON object (parseable by any JSON parser)
- And: no other text appears on stdout

### CG-OUT-02: JSON success schema

- Given: all checks pass
- When: JSON output is produced
- Then: the output matches:

  ```json
  {
    "success": true,
    "mode": null,
    "checks": [
      { "name": "...", "passed": true, "duration_ms": ... }
    ],
    "duration_ms": ...
  }
  ```

- And: `mode` is `null` when no `COMMITGATE_MODE` is active, or the mode name string when a mode is active

### CG-OUT-03: JSON failure schema

- Given: a check fails
- When: JSON output is produced
- Then: the output matches:

  ```json
  {
    "success": false,
    "mode": "coder",
    "checks": [
      {
        "name": "...",
        "passed": false,
        "detail": "...",
        "remediation": "...",
        "duration_ms": ...
      }
    ],
    "duration_ms": ...
  }
  ```

- And: checks skipped due to fail-fast are omitted from the `checks` array
- And: `mode` is `null` when no mode is active, or the mode name string when a mode is active

### CG-OUT-04: Human output includes colour when TTY

- Given: stdout is a TTY
- And: `--json` is not set
- When: the gate produces output
- Then: pass results are rendered in green
- And: fail results are rendered in red

### CG-OUT-05: Non-TTY stdout auto-selects JSON

- Given: stdout is piped (not a TTY)
- And: `--json` is not explicitly set
- When: the gate produces output
- Then: output is JSON format (same as `--json`)

---

## Exit Codes

### CG-EXIT-01: Exit 0 when all checks pass

- Given: every check in the pipeline passes
- When: the gate completes
- Then: the process exits with code 0

### CG-EXIT-02: Exit 1 when any check fails

- Given: at least one check fails
- When: the gate completes
- Then: the process exits with code 1

### CG-EXIT-03: Exit 2 on gate error

- Given: the gate encounters an unexpected error (bad config, missing binary)
- When: the gate cannot complete normally
- Then: the process exits with code 2
- And: if running in human output mode, stderr contains a plain-text error message
- And: if running in JSON mode (`--json` or non-TTY stdout), stderr contains a single JSON object `{"error": "..."}` with the error description
- Note: gate errors are always written to stderr, never stdout, regardless of output mode

---

## Hook Installation

### CG-INST-01: `commitgate install` creates a pre-commit hook

- Given: running `commitgate install` in a git repository
- When: the command succeeds
- Then: a `pre-commit` hook script exists in the configured hook directory
- And: the hook script invokes `commitgate check --staged`
- And: `git config core.hooksPath` points to the hook directory

### CG-INST-02: `commitgate install --commit-msg` creates a commit-msg hook

- Given: running `commitgate install --commit-msg` in a git repository
- When: the command succeeds
- Then: a `commit-msg` hook script exists in the configured hook directory
- And: the hook script invokes `commitgate check --message "$(cat $1)"` (Unix) or equivalent (Windows)
- And: pre-commit and commit-msg hooks coexist without conflict

### CG-INST-03: `commitgate install` fails gracefully outside a git repo

- Given: running `commitgate install` in a directory with no git repository
- When: the command runs
- Then: it exits with code 2
- And: a clear error message states that a git repository is required

### CG-INST-04: `commitgate install --force` overwrites an existing hook

- Given: a pre-commit hook already exists in the hook directory
- When: `commitgate install --force` is run
- Then: the existing hook is replaced with the commitgate hook script
- And: the original hook content is lost (user is responsible for backing it up)

---

## Init

### CG-INIT-01: `commitgate init` creates a valid config file

- Given: no `.commitgate.toml` exists in the current directory
- When: `commitgate init --language rust` is run
- Then: a `.commitgate.toml` is created in the current directory
- And: the generated file is valid TOML parseable by the gate without error
- And: the generated `sequence` includes checks appropriate for Rust (e.g. formatting, clippy, tests)

### CG-INIT-02: `commitgate init` does not overwrite existing config without `--force`

- Given: a `.commitgate.toml` already exists
- When: `commitgate init` is run without `--force`
- Then: it exits with code 1 and an error message identifying the existing file
- And: the existing config is unchanged

### CG-INIT-03: `commitgate init --with-modes` includes mode definitions

- Given: no `.commitgate.toml` exists
- When: `commitgate init --with-modes` is run
- Then: the generated config includes at least two mode entries (e.g. `[modes.coder]` and `[modes.tester]`)
- And: a `[modes._global]` section is present with `always_writable` defaults

---

## Verbose Output

### CG-OUT-06: `--verbose` shows passing check details

- Given: two checks pass and one fails
- And: `--verbose` flag is set
- When: human output is produced
- Then: passing checks include a detail line showing what was checked (e.g. file count or mode name)
- And: checks skipped due to fail-fast are listed with a `skipped` indicator
- And: per-check timing is shown for all executed checks

### CG-OUT-07: `--verbose` does not affect JSON output

- Given: `--verbose` and `--json` are both set
- When: the gate produces output
- Then: output is identical to `--json` alone (`--verbose` adds no extra JSON fields)

---

## MCP Tool (`propose_commit`)

### CG-MCP-01: `propose_commit` runs the pipeline and returns GateResult on failure
- Given: the MCP server is running (`commitgate mcp`)
- And: a `propose_commit` tool call is received with a valid file list and message
- And: a check in the pipeline fails
- When: the tool call is processed
- Then: the tool response is a `GateResult` JSON object with `success: false`
- And: the response includes check results with `detail` and `remediation` fields
- And: no git commit is executed
- And: the MCP response exit code is a tool-result (not an MCP error)

### CG-MCP-02: `propose_commit` commits and returns success when all checks pass
- Given: the MCP server is running
- And: a `propose_commit` tool call is received with a valid file list and message
- And: all checks pass
- When: the tool call is processed
- Then: `git commit -m <message>` is executed with the provided message
- And: the tool response is a `GateResult` JSON object with `success: true`
- And: the commit is present in the repository's git history

### CG-MCP-03: `propose_commit` returns an MCP error on gate error (not a check failure)
- Given: the MCP server is running
- And: a `propose_commit` tool call is received
- And: the gate cannot run (e.g. no `.commitgate.toml`, missing tool)
- When: the tool call is processed
- Then: the response is an MCP protocol error (not a `GateResult`)
- And: the error describes the configuration or environment problem
- And: no git commit is executed

### CG-MCP-04: `propose_commit` does not commit unless pipeline explicitly succeeds
- Given: the pipeline returns a gate error (code 2 equivalent)
- When: the MCP server processes the result
- Then: git commit is NOT called under any circumstance
- Note: this is distinct from a check failure — it tests that the MCP handler has no code path that calls git commit on a non-success result

### CG-MCP-05: MCP server is usable via stdio
- Given: `commitgate mcp` is launched as a subprocess
- And: a valid MCP `initialize` request is sent on stdin
- When: the server processes the request
- Then: a valid MCP `initialize` response is written to stdout
- And: the `tools` list includes `propose_commit` with its input schema
