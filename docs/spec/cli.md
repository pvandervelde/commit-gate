# CLI Interface

## Commands

### `commitgate check`

Run the check pipeline against staged files or a specified file list.

```
commitgate check [OPTIONS]

Options:
  --staged           Check files in the git staging area (default when no --files).
                     Mutually exclusive with --files. Providing both is a gate error (exit 2).
  --files <FILE>...  Check specific files from the filesystem (working tree). Note: unlike --staged,
                     this reads file content directly from disk, not from the git index. Useful for
                     pre-flight checks before staging. Mutually exclusive with --staged.
                     When COMMITGATE_MODE is active, path restrictions apply to the specified files
                     just as they do for --staged. This makes --files useful for validating a file
                     list before committing it.
  --message <MSG>    Commit message to validate (for message check). Only relevant when the message
                     check is in the pipeline. When running as a pre-commit hook this flag is not
                     available — use the commit-msg hook integration instead (see `commitgate install`).
  --mode <MODE>      Override COMMITGATE_MODE environment variable
  --config <PATH>    Override config file discovery (use specific path)
  --json             Force JSON output (default when stdout is not a TTY)
  --no-color         Disable coloured output (also respects NO_COLOR env var)
  --verbose          Show passing checks and timing details
  -h, --help         Print help
```

**Exit codes:**

- `0` — All checks passed
- `1` — At least one check failed
- `2` — Gate error (bad config, missing tool, not in a git repo)

**Examples:**

```bash
# Check staged files (typical pre-commit usage)
commitgate check --staged

# Check specific files (dry-run before staging)
commitgate check --files src/auth.rs src/session.rs

# Check with mode override
commitgate check --staged --mode tester

# JSON output for agent consumption
commitgate check --staged --json

# Validate commit message
commitgate check --staged --message "feat(auth): Add rate limiting"
```

### `commitgate init`

Scaffold a `.commitgate.toml` file with sensible defaults.

```
commitgate init [OPTIONS]

Options:
  --language <LANG>  Preset for language-specific checks (rust, typescript, python)
  --with-modes       Include mode definitions for agent role separation
  --force            Overwrite existing .commitgate.toml
  -h, --help         Print help
```

**Examples:**

```bash
# Create config with Rust defaults
commitgate init --language rust

# Create config with modes for agent separation
commitgate init --language rust --with-modes
```

### `commitgate install`

Install the gate as a git hook.

```
commitgate install [OPTIONS]

Options:
  --hook-dir <DIR>   Directory for hooks (default: .githooks)
  --global           Install globally for all repos via git config
  --commit-msg       Also install a commit-msg hook to validate the commit message.
                     The commit-msg hook invokes `commitgate check --message "$(cat $1)"`.
                     Use this when the message check is in your pipeline sequence.
  --force            Overwrite existing hook scripts
  -h, --help         Print help
```

This creates the hook directory, writes the pre-commit hook script that invokes `commitgate check --staged`, and configures `git config core.hooksPath`. When `--commit-msg` is also specified, a `commit-msg` hook is created alongside the `pre-commit` hook.

**When to use `--commit-msg`:** The pre-commit hook runs _before_ the commit message is entered by the developer. The message check can only validate the final message in the commit-msg hook, which runs _after_ the developer has typed the message. If your `sequence` includes a `message` check, install both hooks.

**Examples:**

```bash
# Install in current repo
commitgate install

# Install with custom hook directory
commitgate install --hook-dir .git/hooks
```

### `commitgate --version`

Print the binary version and exit.

### `commitgate --help`

Print the top-level help and exit.

### `commitgate mcp`

Start a stdio MCP server. The server exposes the `propose_commit` tool for use by MCP-compatible AI agents (Claude Code and equivalents).

```
commitgate mcp [OPTIONS]

Options:
  --config <PATH>    Override config file discovery (use specific path)
  -h, --help         Print help
```

The server reads from stdin and writes to stdout using the MCP JSON-RPC protocol. It should be launched as a subprocess by the agent framework, not run interactively.

**Exposed tool: `propose_commit`**

```
Input:
  files:   string[]   — file paths to validate and stage
  message: string     — commit message to use on success

Output (GateResult):
  success:     bool
  mode:        string | null
  checks:      CheckResult[]
  duration_ms: int
```

When `success` is `true` the server has already executed `git commit -m <message>`. The agent should not call `git commit` separately.

**Example MCP server config (Claude Code):**

```json
{
  "mcpServers": {
    "commitgate": {
      "command": "commitgate",
      "args": ["mcp"]
    }
  }
}
```

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `COMMITGATE_MODE` | Active mode name. Enables path restrictions for the named mode. |
| `NO_COLOR` | When set (any value), disables coloured output. See <https://no-color.org/> |

---

## Output Formats

### Human Output (default when stdout is a TTY)

```
━━━ commitgate ━━━
✓ paths (tester)                    0ms
✗ no-stubs                          12ms
  src/auth.rs:42: todo!()
  src/session.rs:7: unimplemented!()
  → Replace all stub markers with actual implementations before committing.
━━━━━━━━━━━━━━━━━━
BLOCKED: 1 of 3 checks failed (fail-fast: 2 checks skipped)
```

With `--verbose`:

```
━━━ commitgate ━━━
✓ paths (tester)                    0ms
  Checked 3 files against writable paths: tests/
✓ formatting                        340ms
✗ no-stubs                          12ms
  src/auth.rs:42: todo!()
  src/session.rs:7: unimplemented!()
  → Replace all stub markers with actual implementations before committing.
— clippy                            skipped (fail-fast)
— tests                             skipped (fail-fast)
━━━━━━━━━━━━━━━━━━
BLOCKED: 1 of 5 checks failed. Total: 352ms
```

### JSON Output (default when stdout is not a TTY, or `--json`)

**Success:**

```json
{
  "success": true,
  "mode": null,
  "checks": [
    { "name": "formatting", "passed": true, "duration_ms": 340 },
    { "name": "no-stubs", "passed": true, "duration_ms": 12 },
    { "name": "clippy", "passed": true, "duration_ms": 2100 },
    { "name": "tests", "passed": true, "duration_ms": 4500 }
  ],
  "duration_ms": 6952
}
```

**Failure:**

```json
{
  "success": false,
  "mode": "coder",
  "checks": [
    { "name": "paths", "passed": true, "duration_ms": 0 },
    { "name": "formatting", "passed": true, "duration_ms": 340 },
    {
      "name": "no-stubs",
      "passed": false,
      "detail": "src/auth.rs:42: todo!()\nsrc/session.rs:7: unimplemented!()",
      "remediation": "Replace all stub markers with actual implementations before committing.",
      "duration_ms": 12
    }
  ],
  "duration_ms": 352
}
```

**Gate error (exit code 2, written to stderr):**

```json
{
  "error": "Configuration error: unknown check type 'builtin-v2' in check 'paths' (.commitgate.toml line 12)"
}
```

Checks that were skipped due to fail-fast are omitted from the `checks` array. The consumer can infer skipped checks by comparing the array against the configured sequence.
