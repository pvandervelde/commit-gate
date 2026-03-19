# Domain Vocabulary

## Core Concepts

### Commit Gate

The system that validates a proposed commit against a pipeline of checks before allowing it into git history. Acts as a gatekeeper between working tree changes and permanent history.

### Check

A single validation step in the pipeline. Each check examines the proposed commit (staged files, commit message, or both) and returns a pass/fail result with optional diagnostic detail. Checks are either built-in (compiled into the binary) or command-based (shell commands invoked by the gate).

### Check Pipeline

An ordered sequence of checks executed against a proposed commit. Checks run in the order defined in configuration. In fail-fast mode (default), the first failure halts the pipeline. In run-all mode, every check runs regardless of prior failures.

### Check Result

The outcome of a single check execution. Contains: the check name, pass/fail status, diagnostic detail (on failure), a human-readable remediation instruction, execution duration, and truncated output. A check result is always structured and machine-parseable, even in human-readable output mode.

### Gate Result

The aggregate outcome of running the full check pipeline against a proposed commit. Contains: overall pass/fail, the list of individual check results, and timing information. The gate never creates a commit, so it never produces a commit hash — that is git's responsibility after the gate exits 0.

### Mode

An optional named configuration that restricts which file paths are permitted in a commit. Used for agent role separation (e.g. a "tester" mode only allows commits to `tests/`). When no mode is active, path restrictions are not enforced.

### Configuration File

The `.commitgate.toml` file at the repository root. Single source of truth for all check definitions, mode definitions, and pipeline settings. Discovered by walking up from the current working directory.

### Remediation

An imperative instruction attached to a failed check result that tells the developer (or agent) exactly what to do to fix the failure. Must be complete and actionable without additional context. Examples: "Run `cargo fmt` to fix formatting", "Replace `todo!()` on line 42 of src/auth.rs with actual implementation".

## Check Types

### Built-in Check: paths

Validates that all staged files fall within the allowed paths for the active mode. Always runs first when a mode is active. No-ops when no mode is set. Configured with `type = "paths"` in `.commitgate.toml`.

### Built-in Check: no-stubs

Scans staged file contents for configurable patterns that indicate incomplete implementation (e.g. `todo!()`, `// TODO`, `unimplemented!()`). Reports the file, line number, and matched pattern for each hit.

### Built-in Check: message

Validates the commit message against configurable rules: minimum length, conventional commit format, forbidden patterns (e.g. "WIP", "fixup").

### Command Check

A check defined as a shell command template. The gate executes the command, captures stdout and stderr, and interprets the exit code: 0 = pass, non-zero = fail. Output is truncated to a configurable line limit.

## Output Concepts

### Human Output

Coloured terminal output with check names, pass/fail indicators, and diagnostic detail. Used when stdout is a TTY and `--json` is not set.

### JSON Output

A single JSON object written to stdout containing the complete gate result. Used when `--json` is set or stdout is not a TTY. Designed for machine consumption by AI coding agents.

### Output Truncation

Check output (stdout + stderr from command checks, or match lists from built-in checks) is capped at a configurable number of lines per check. This prevents flooding the agent's context window with thousands of lines of test output. The truncation limit is configurable per-check and globally.

## Error Concepts

### Check Failure

An expected outcome: a check ran successfully but the proposed commit did not satisfy its criteria. Returns structured diagnostic information. Exit code 1 from the gate (with `success: false` in JSON).

### Gate Error

An unexpected problem: the gate itself could not execute (missing config, malformed TOML, binary not found, check command not installed). Returns a human-readable error message. Exit code non-zero from the gate.

### Staged File

A file that has been `git add`-ed and is part of the next commit. The gate operates on staged content (via `git show :path`), not working tree content. This ensures the gate validates exactly what will be committed.

## Agent Integration Concepts

### MCP Tool

A structured tool interface exposed by `commitgate mcp` (a stdio MCP server). AI agents that support the Model Context Protocol call `propose_commit` as a first-class tool instead of shelling out to `git commit`. The tool runs the full check pipeline and, on success, executes the commit. The agent receives a `GateResult` as the tool return value.

### `propose_commit`

The MCP tool name exposed by the gate's MCP server. Accepts `files: string[]` and `message: string`. Staging lifecycle: (1) stages the provided files with `git add`; (2) runs the full check pipeline against the staged index; (3a) on success, executes `git commit -m <message>` and returns `GateResult { success: true }`; (3b) on failure or gate error, executes `git restore --staged <files>` to restore the staging area to its pre-call state and returns `GateResult { success: false }` or an MCP error. The repository staging area is always left unchanged relative to its state before the call when the pipeline does not succeed.

### Git Alias Wrapper

A shell script named `git` placed earlier in `PATH` than the real git binary in an agent's execution environment. Intercepts `git commit` subcommands and runs `commitgate check --staged --json` before delegating to real git. Transparent to the agent — the agent cannot distinguish the wrapper from real git. Not for use on human developer machines.

### Copilot Skill

A GitHub Copilot Extension or skill definition (`.copilot/skills/commitgate.yaml`) that wraps `commitgate check` as an invocable action for Copilot-based agents. Serves the same purpose as the MCP tool for the Copilot agent framework.
