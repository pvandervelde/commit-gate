# Overview

## System Context

Commit Gate sits between the developer (or AI agent) and git history. It intercepts commit operations and validates them before they reach the repository.

```
Developer / AI Agent
        │
        │  git commit  (or: commitgate check --staged)
        │
        ▼
┌───────────────────┐
│   Commit Gate     │
│                   │
│  ┌─────────────┐  │     ┌──────────────────┐
│  │ Config      │◄─┼─────│ .commitgate.toml │
│  │ Loader      │  │     └──────────────────┘
│  └──────┬──────┘  │
│         │         │
│  ┌──────▼──────┐  │
│  │ Pipeline    │  │     ┌──────────────────┐
│  │ Runner      │──┼────►│ External tools   │
│  │             │  │     │ (cargo fmt, etc) │
│  └──────┬──────┘  │     └──────────────────┘
│         │         │
│  ┌──────▼──────┐  │
│  │ Output      │  │
│  │ Formatter   │  │
│  └──────┬──────┘  │
└─────────┼─────────┘
          │
          ▼
   pass → git commit executes
   fail → structured feedback, no commit
```

## Integration Points

### As a Git Pre-Commit Hook

Primary integration for human developers. The binary is invoked by git before every commit. It reads the staged files, runs the check pipeline, and exits 0 (allow commit) or 1 (reject commit). Installed via `git config core.hooksPath` pointing to a directory containing the gate as `pre-commit`.

**Known limitations for agent use:**

- **Per-repo scope.** Git hooks are configured per repository (or globally per user machine). Every repository where the gate should apply must be configured. In an agent environment with many ephemeral repositories, this requires provisioning the hooks as part of the environment setup.
- **Bypassable.** `git commit --no-verify` skips all hooks entirely. The gate has no mechanism to prevent or detect this. An agent that uses `--no-verify` (deliberately or by mistake) bypasses all checks with no feedback.
- **Poor feedback channel.** Hook output travels through git's hook invocation pipe. Some agent integrations (those that shell out to `git commit` and only inspect the exit code) miss the hook's stdout/stderr entirely. The agent sees "commit failed" with no structured reason. This is a fundamental limitation of the git hook model, not of the gate's output format.

### As a Git Alias Wrapper (for agent environments)

A wrapper script named `git` is placed earlier in `PATH` than the real git binary. The wrapper intercepts `git commit` subcommands, runs `commitgate check --staged --json` first, and only delegates to real git if all checks pass. All other git subcommands pass straight through.

**Advantages over hooks:**

- Not bypassable via `--no-verify` — the agent calls `git commit` normally and has no visible way to circumvent the gate
- Not per-repo — the wrapper lives in the environment (container image, sandbox PATH) and applies to all repositories without any per-repo setup
- Structured feedback — the wrapper captures the gate's JSON output and can re-surface it clearly, avoiding the feedback-loss problem of hook pipes

**Limitations:**

- Requires controlling the agent's execution environment (PATH, container image)
- A bug in the wrapper can block all commits; the real git binary must be reachable as a fallback
- Transparent to the agent — the agent cannot distinguish the wrapper from real git, which may make debugging agent behaviour harder
- Does not apply to human developer machines (not appropriate for interactive use)

See [tradeoffs.md](tradeoffs.md) T-07 for full analysis.

### As a Standalone CLI

Developers (and scripts) run `commitgate check --staged` to validate before committing, or `commitgate check --files src/main.rs src/lib.rs` to check specific files. Useful for dry-run validation and CI pipelines.

### As an MCP Tool (Claude Code / MCP-compatible agents)

The gate exposes a `propose_commit` MCP tool via `commitgate mcp` (stdio server). The tool accepts a file list and commit message, runs the full check pipeline, and returns a structured `GateResult`. The agent calls `propose_commit` rather than shelling out to `git commit`, giving it a first-class feedback interface.

**Advantages over hooks:**

- Agent receives full `GateResult` JSON as the tool's return value — no output-capture problem
- Explicit: the agent knows it is going through a gate; it cannot accidentally bypass it
- Works across repositories without per-repo hook setup, as long as the MCP server is running
- Remediation instructions are returned as structured data, making autonomous self-correction straightforward

**Limitations:**

- Requires the agent to be MCP-aware and configured to use the `propose_commit` tool
- The MCP server process must be running; adds operational overhead
- Not applicable to non-MCP agent frameworks without adaptation

See [tradeoffs.md](tradeoffs.md) T-07 for full analysis.

### As a GitHub Copilot Skill

A Copilot skill definition (`.copilot/skills/commitgate.yaml`) wraps the `commitgate check` CLI as an invocable action. When Copilot wants to commit code it calls the skill, receives the structured JSON result, and either proceeds or self-corrects. This is the Copilot-native equivalent of the MCP tool integration.

**Relationship to MCP tool:** The skill definition and MCP tool serve the same purpose for different agent frameworks. If both Claude Code and Copilot are in use, both integrations can be provided — they invoke the same underlying `commitgate check` binary.

### As a Commit-Msg Hook

Optional secondary hook. When the `message` check is configured, the gate runs as a `commit-msg` hook to validate the commit message after the developer has typed it. This is the only supported way to validate commit messages when running as a git hook (the pre-commit hook runs before the message is entered).

## Operational Model

### Normal Operation (pre-commit hook)

1. Developer stages files with `git add`
2. Developer runs `git commit -m "message"`
3. Git invokes the pre-commit hook (Commit Gate)
4. Gate discovers `.commitgate.toml` by walking up from cwd
5. Gate reads `COMMITGATE_MODE` env var (if set, enables path restrictions)
6. Gate reads the list of staged files from git
7. Gate runs each check in the configured sequence
8. On first failure (fail-fast mode): gate prints diagnostic, exits 1, commit aborted
9. On all pass: gate exits 0, commit proceeds

### Agent Operation (JSON mode)

Same as above, but:

- Output is structured JSON (auto-detected when stdout is not a TTY, or via `--json`)
- The `action` field in each failure tells the agent exactly what to fix
- The agent can re-attempt after making corrections (autonomous self-correction loop)

### CI Operation (standalone CLI)

- `commitgate check --staged` in a CI pipeline validates the same rules
- Exit code 0 = pass, non-zero = fail
- JSON output can be parsed by CI tooling for structured reporting
- Same `.commitgate.toml`, same checks, same results as local development

## External Dependencies

Commit Gate itself has no runtime dependencies (single static binary). However, the command checks it invokes depend on external tools being installed:

| Check | Requires | Typical Install |
|---|---|---|
| formatting (Rust) | `cargo fmt` | Ships with rustup |
| formatting (TS) | `prettier` | `npm install -g prettier` |
| formatting (Python) | `black` or `ruff` | `pip install black` |
| clippy | `cargo clippy` | Ships with rustup |
| tests (Rust) | `cargo test` | Ships with rustup |
| tests (TS) | `jest` / `vitest` | `npm install` |
| tests (Python) | `pytest` | `pip install pytest` |

If a command check's tool is not installed, the check fails with a clear error message naming the missing tool. This is a gate error (unexpected), not a check failure (expected).
