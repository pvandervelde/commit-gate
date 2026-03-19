# Implementation Constraints

## Language and Build

- Written in Rust, edition 2021 or later
- Compiles to a single static binary with no runtime dependencies
- Cross-compile targets: `x86_64-unknown-linux-musl`, `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-pc-windows-msvc`
- Binary must be usable without installation — download, `chmod +x`, run
- Binary startup time must be under 100ms (excluding check execution)
- Binary must print version via `commitgate --version`

## Type System

- All domain identifiers use Rust newtypes where meaningful (e.g. `CheckName(String)`, `Mode(String)`)
- All fallible operations return `Result<T, E>` — no `unwrap()` or `expect()` in library code (`main.rs` may use them at the top level after error conversion)
- Error types use enums with descriptive variants
- Configuration parsing errors carry source location (line/column)

## Error Handling

- Gate errors (bad config, missing tool) exit with code 2 and an error message on stderr
  - In human output mode: plain text
  - In JSON mode (`--json` or non-TTY stdout): a JSON object `{"error": "..."}` on stderr
- Check failures exit with code 1 and structured output on stdout (human or JSON)
- The gate never panics in normal operation — all error paths are handled explicitly
- Configuration errors are reported before any check executes
- Missing external tools are reported with the tool name and a suggestion for installation

## Output

- JSON output to stdout must be a single, complete JSON object — no streaming, no multiple objects, no trailing newlines after the closing brace
- Human output must be readable in a standard 80-column terminal
- Colour output must respect the `NO_COLOR` environment variable (see <https://no-color.org/>)
- Output truncation (`max_output_lines`) applies to **failing** checks only. Passing checks show brief status text (a single line), not full stdout/stderr, even in verbose mode. In `--verbose` mode, a passing command check shows a single summary line (e.g. "exit 0, 340ms") — the check's captured stdout/stderr is discarded. There is no mode that shows the full stdout of a passing command check.
- Output truncation must clearly indicate how many lines were omitted
- The `remediation` field in every failed check must be a complete, imperative sentence that the developer or agent can act on without additional context

## Process Execution

- Command checks are executed as child processes via the system shell (`sh -c` on Unix, `cmd /C` on Windows)
- Each command check has an independent timeout (configurable per-check, with a global default)
- Timed-out processes are killed with SIGKILL (Unix) or TerminateProcess (Windows) after a grace period
- stdout and stderr from command checks are captured separately but merged in the diagnostic output
- The `{files}` template in command strings is replaced with space-separated staged file paths, each shell-escaped

## Git Integration

- The gate reads from the git staging area only — never the working tree — when invoked via `--staged` or as a git hook
- When invoked via `--files`, the gate reads file content directly from the filesystem (working tree). The git staging area is not used for content-based checks in this mode.
- `--staged` and `--files` are mutually exclusive. Providing both is a gate error (exit code 2).
- Staged file content is read via `git show :<path>`, not by reading the filesystem
- The `commitgate check` command never writes to git (no `git add`, `git commit`, `git reset`, or any write operations). The `commitgate mcp` server does write to git as part of the `propose_commit` operation — see security.md for the constrained write surface.
- The git binary is assumed to be in PATH. If not found, the gate exits with code 2 and a clear error

## Configuration

- `.commitgate.toml` is discovered by walking up from the current working directory, stopping at the filesystem root
- The file must be valid TOML. Partial or streaming parsing is not supported
- Unknown keys in the TOML file are ignored (forward compatibility)
- Unknown check types are rejected (backward compatibility — don't silently skip checks). Valid built-in `type` values are: `"paths"`, `"pattern-scan"`, `"message"`, `"command"`
- The `sequence` array in `[checks]` determines execution order. Checks not in the sequence are not executed, even if defined
- References in `sequence` to check names that have no corresponding `[checks.<name>]` section are rejected with a gate error (code 2) before any checks run
- The `modes` section is optional. Omitting it entirely disables path restrictions regardless of `COMMITGATE_MODE`

## Performance

- The gate itself (excluding external check commands) must complete in under 1 second for a typical repository with 50 staged files
- File content reads from git should be batched where possible (single `git show` per file, not per pattern)
- Pattern scanning uses byte-level string search (not regex) for stub detection — patterns are literal strings

## Compatibility

- Minimum supported git version: 2.20 (for `--diff-filter=ACMR` support)
- Works in shallow clones and worktrees
- Works when invoked from any directory within the repository (not just the root)
- Config file path can be overridden via `--config` flag (for testing and CI)
- The `.commitgate.toml` format is a strict subset of `.agentgate.toml` — any valid `.commitgate.toml` is also valid as the relevant sections of `.agentgate.toml`
