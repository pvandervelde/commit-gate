# Configuration Reference

## Example `.commitgate.toml`

```toml
# .commitgate.toml — Commit quality gate configuration
#
# This file is the single source of truth for all check definitions,
# mode definitions, and pipeline settings.
#
# Format is a strict subset of .agentgate.toml — if you later adopt
# agentgate, this config can be merged into it without changes.

# ─────────────────────────────────────────────────────────────────────
# CHECKS — quality gate pipeline
# ─────────────────────────────────────────────────────────────────────

[checks]
# Ordered list of check names to execute. Checks not listed here are
# defined but not executed (useful for temporarily disabling a check).
sequence = ["paths", "formatting", "no-stubs", "clippy", "tests"]

# Stop on first failure (default: true). Set to false to run all
# checks and report all failures at once.
fail_fast = true

# Default maximum output lines per check (default: 30).
# Prevents flooding terminal or agent context window.
max_output_lines = 30

# Default timeout in seconds for command checks (default: 60).
default_timeout = 60


# ── Built-in: path restriction ──────────────────────────────────────
[checks.paths]
type = "paths"
# Active only when COMMITGATE_MODE is set.
# Validates staged files against the mode's writable paths.

# ── Built-in: stub pattern scan ─────────────────────────────────────
[checks.no-stubs]
type = "pattern-scan"
patterns = [
    "todo!()",
    "unimplemented!()",
    'panic!("not implemented")',
    "// TODO",
    "// FIXME",
    "// HACK",
    "// XXX",
]
# Only scan files with these extensions. Omit to scan all text files.
extensions = [".rs"]
remediation = "Replace all stub markers with actual implementations before committing."

# ── Built-in: commit message validation ─────────────────────────────
[checks.message]
type = "message"
min_length = 15
# Options: "conventional", "none"
format = "conventional"
# Patterns that cause the message check to fail
forbidden_patterns = ["WIP", "fixup!", "squash!"]
remediation = "Write a descriptive commit message in conventional format: <type>(<scope>): <subject>"

# ── Command: formatting ─────────────────────────────────────────────
[checks.formatting]
type = "command"
command = "cargo fmt --check"
remediation = "Run 'cargo fmt' to fix formatting, then re-stage the files."
# Override default timeout for this check (seconds)
# timeout = 30

# ── Command: clippy ─────────────────────────────────────────────────
[checks.clippy]
type = "command"
command = "cargo clippy -- -D warnings"
remediation = "Fix all clippy warnings before committing."
# Override max output lines for this check
# max_output_lines = 50

# ── Command: tests ──────────────────────────────────────────────────
[checks.tests]
type = "command"
command = "cargo test"
timeout = 120
remediation = "All tests must pass before committing. Fix failing tests."


# ─────────────────────────────────────────────────────────────────────
# MODES — optional path restrictions for agent role separation
# ─────────────────────────────────────────────────────────────────────
# Modes are activated via the COMMITGATE_MODE environment variable.
# When no mode is active, the path check is skipped.
# This entire section is optional — omit it if you don't use modes.

[modes.tester]
writable = ["tests"]
# ADVISORY ONLY — the gate does NOT enforce readonly.
# This field is documentation for humans; the gate only enforces 'writable'.
# If you want path restrictions, adjust 'writable', not 'readonly'.
readonly = ["src", "lib"]

[modes.coder]
writable = ["src", "lib"]
# ADVISORY ONLY — see note above
readonly = ["tests"]

# Paths that are always writable regardless of mode.
# Default: ["docs", ".llm"]
[modes._global]
always_writable = ["docs", ".llm", ".commitgate.toml"]
```

## Schema Reference

### `[checks]`

| Field | Type | Default | Description |
|---|---|---|---|
| `sequence` | `string[]` | required | Ordered list of check names to execute |
| `fail_fast` | `bool` | `true` | Halt pipeline on first failure |
| `max_output_lines` | `int` | `30` | Default output truncation per check |
| `default_timeout` | `int` | `60` | Default timeout (seconds) for command checks |

### `[checks.<name>]` — Built-in: path restriction

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `"paths"` | required | Must be `"paths"` |

No additional fields. Behaviour is controlled by the active mode.

### `[checks.<name>]` — Built-in: pattern scan

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `"pattern-scan"` | required | Must be `"pattern-scan"` |
| `patterns` | `string[]` | required | Exact byte sequences to search for in staged file content. **Not regular expressions** — patterns are matched literally. Special regex characters (`.`, `*`, `?`, `[`, etc.) have no special meaning and are matched as-is. |
| `extensions` | `string[]` | all files | File extensions to scan (e.g. `[".rs", ".ts"]`) |
| `remediation` | `string` | `""` | Action instruction on failure |

### `[checks.<name>]` — Built-in: commit message

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `"message"` | required | Must be `"message"` |
| `min_length` | `int` | `10` | Minimum commit message subject length |
| `format` | `string` | `"none"` | Message format: `"conventional"` (follows [Conventional Commits v1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — `<type>(<scope>): <subject>`, where `type` is a standard verb such as `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `style`, `perf`, `ci`; scope is optional) or `"none"` |
| `forbidden_patterns` | `string[]` | `[]` | Patterns that cause the check to fail |
| `remediation` | `string` | `""` | Action instruction on failure |

### `[checks.<name>]` — Command check

| Field | Type | Default | Description |
|---|---|---|---|
| `type` | `"command"` | required | Must be `"command"` |
| `command` | `string` | required | Shell command to execute. `{files}` is replaced with staged file paths |
| `timeout` | `int` | inherits `default_timeout` | Per-check timeout in seconds |
| `max_output_lines` | `int` | inherits global | Per-check output truncation |
| `remediation` | `string` | `""` | Action instruction on failure |

### `[modes.<name>]`

| Field | Type | Default | Description |
|---|---|---|---|
| `writable` | `string[]` | required | Directory prefixes this mode can commit to |
| `readonly` | `string[]` | `[]` | Advisory only — directories this mode should not write to. **No runtime enforcement.** Serves as documentation for humans reviewing the config. The gate enforces only `writable` paths. |

### `[modes._global]`

| Field | Type | Default | Description |
|---|---|---|---|
| `always_writable` | `string[]` | `["docs", ".llm"]` | Paths writable in all modes |
