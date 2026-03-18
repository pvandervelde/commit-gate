# Operations

## Distribution

Commit Gate is distributed as a pre-built static binary. No installer, no runtime dependencies.

### Release Artefacts

Each release publishes binaries for the following targets:

| Target | Platform |
|---|---|
| `x86_64-unknown-linux-musl` | Linux x86-64 (musl, fully static) |
| `aarch64-unknown-linux-musl` | Linux ARM64 (musl, fully static) |
| `aarch64-apple-darwin` | macOS Apple Silicon |
| `x86_64-apple-darwin` | macOS Intel |
| `x86_64-pc-windows-msvc` | Windows x86-64 |

Binaries are published as GitHub Release assets and as a Homebrew formula (macOS/Linux).

### Installation Methods

**Manual (all platforms):**

```bash
# Download the binary for your platform, make it executable, place it in PATH
curl -L https://github.com/pvandervelde/commit-gate/releases/latest/download/commitgate-x86_64-unknown-linux-musl \
  -o /usr/local/bin/commitgate
chmod +x /usr/local/bin/commitgate
```

**Homebrew (macOS/Linux):**

```bash
brew install pvandervelde/tap/commitgate
```

**Cargo (requires Rust toolchain):**

```bash
cargo install commitgate
```

---

## Upgrading

### Minor and Patch Releases

Minor and patch releases are backwards-compatible. The `.commitgate.toml` format does not change in breaking ways. Upgrade by replacing the binary.

To check the current version:

```bash
commitgate --version
```

### Major Releases

Major releases may introduce breaking changes to the config format. Release notes will document migration steps. The gate will emit a clear error if it encounters an old config key that has been removed or renamed, rather than silently ignoring it.

### Version Check in CI

To pin a specific version in CI:

```yaml
- name: Install commitgate
  run: |
    curl -L https://github.com/pvandervelde/commit-gate/releases/download/v1.2.3/commitgate-x86_64-unknown-linux-musl \
      -o /usr/local/bin/commitgate
    chmod +x /usr/local/bin/commitgate
```

---

## Agent Integration

This section covers deployment patterns for AI agent environments. See [tradeoffs.md](tradeoffs.md) T-07 for the full rationale and comparison of approaches.

### Option B: Git Alias Wrapper (recommended for containerised agents)

Place a wrapper script named `git` earlier in `PATH` than the real git binary. The wrapper intercepts `git commit`, runs the gate first, and delegates all other commands to real git.

**Dockerfile example:**
```dockerfile
# Install commitgate
RUN curl -L https://github.com/pvandervelde/commit-gate/releases/latest/download/commitgate-x86_64-unknown-linux-musl \
      -o /usr/local/bin/commitgate && chmod +x /usr/local/bin/commitgate

# Install git wrapper (must be earlier in PATH than real git, e.g. /usr/local/bin)
COPY git-wrapper.sh /usr/local/bin/git
RUN chmod +x /usr/local/bin/git
# Real git is at /usr/bin/git — the wrapper calls it directly
```

**`git-wrapper.sh`:**
```bash
#!/usr/bin/env bash
# Git wrapper — intercepts 'git commit' and enforces commitgate

REAL_GIT=/usr/bin/git   # absolute path to real git binary

case "$1" in
  commit)
    OUTPUT=$(commitgate check --staged --json 2>&1)
    STATUS=$?
    if [ $STATUS -ne 0 ]; then
      echo "$OUTPUT" >&2
      exit 1
    fi
    exec "$REAL_GIT" "$@"
    ;;
  *)
    exec "$REAL_GIT" "$@"
    ;;
esac
```

**Key points:**
- Use an **absolute path** to real git (`/usr/bin/git`) in the wrapper, not a PATH lookup, to avoid infinite recursion
- Write gate output to **stderr** in the wrapper so it appears alongside the agent's error context
- The wrapper handles only `commit`; all other subcommands (`status`, `add`, `push`, `log`, etc.) pass through unchanged
- Do **not** install this wrapper on human developer machines

---

### Option C: MCP Tool (Claude Code and MCP-compatible agents)

Run `commitgate mcp` as a stdio MCP server. The agent uses the `propose_commit` tool instead of calling `git commit` directly.

**Claude Code configuration (`.claude/mcp.json` or equivalent):**
```json
{
  "mcpServers": {
    "commitgate": {
      "command": "commitgate",
      "args": ["mcp"],
      "env": {}
    }
  }
}
```

**`propose_commit` tool interface (MCP):**
```
Tool: propose_commit
Input:
  files:   string[]   — paths to stage and check
  message: string     — commit message
Output:
  GateResult JSON — success/failure, check results, remediations
Side-effect on success: git commit is executed by the MCP server
```

**Agent self-correction loop:**
```
1. Agent calls propose_commit({files: [...], message: "feat: ..."})
2. Gate fails → agent receives structured GateResult with remediation per check
3. Agent applies fixes to failing checks
4. Agent calls propose_commit(...) again
5. Gate passes → commit made, success: true returned
```

---

### Option C: GitHub Copilot Skill

A Copilot skill wraps `commitgate check` as an invocable action. Create a skill definition in `.copilot/skills/commitgate.yaml` (format follows the Copilot Extensions / skills specification in use at the time of integration):

```yaml
# .copilot/skills/commitgate.yaml
name: commitgate
description: >
  Run the commit quality gate before committing. Always call this tool
  before executing git commit. Do not use git commit directly.
command: commitgate check --staged --json
workingDirectory: "{{workspace}}"
```

The exact format and registration mechanism follow the GitHub Copilot Extensions documentation in effect at the time of deployment. The gate binary and `.commitgate.toml` must be present in the repository for the skill to function.

---

### Combining Options B and C

For maximum reliability in agent environments, deploy **both**:

- Option B (alias wrapper) in the container: acts as a non-bypassable safety net even if the agent falls back to raw shell commands
- Option C (MCP tool or Copilot skill): provides the preferred commit path with rich structured feedback

The alias wrapper does not interfere with the MCP tool. When the agent uses `propose_commit`, the MCP server calls real git directly (bypassing the wrapper) because the commit only executes after the gate has already passed. The wrapper applies only when the agent calls `git commit` through the shell.

---

## Hook Lifecycle

### Initial Setup (per repository)

```bash
# 1. Download and install the binary
brew install pvandervelde/tap/commitgate

# 2. Scaffold a config file
commitgate init --language rust

# 3. Review and edit .commitgate.toml as needed

# 4. Install the git hooks
commitgate install          # installs pre-commit hook
commitgate install --commit-msg  # also installs commit-msg hook (if using message check)

# 5. Commit the config file
git add .commitgate.toml .githooks/
git commit -m "chore: add commitgate quality gate"
```

### Removing the Gate

```bash
# Remove hook directory from git config
git config --unset core.hooksPath

# Optionally remove the hook directory and config
rm -rf .githooks .commitgate.toml
```

### Global Installation

To install the gate for all future repositories on a machine:

```bash
commitgate install --global
```

This sets `git config --global core.hooksPath` to a user-level hooks directory. The gate must be in PATH for this to work across repositories.

---

## CI Integration

### GitHub Actions

```yaml
- name: Run quality gate
  run: commitgate check --staged --json
```

In CI, `commitgate check --staged` validates the files modified in the current commit. Combined with `--json`, the output can be parsed for structured reporting.

### Exit Code Handling

CI pipelines should treat:

- Exit 0: pass — continue the pipeline
- Exit 1: check failure — fail the step, report structured output
- Exit 2: gate error — fail the step and alert on infrastructure problems (misconfiguration, missing tools)

---

## Monitoring and Observability

The gate writes no logs to disk and has no persistent state. Monitoring is limited to:

- **CI pass/fail rates**: track the frequency of gate failures in CI to identify problematic checks or configurations
- **Check timing**: the JSON output includes `duration_ms` per check and overall; CI tooling can trend these to detect slow checks
- **Agent correction loops**: AI agent pipelines can track how many correction cycles (propose → gate fails → fix → propose again) a typical commit requires

There is no telemetry. No data is sent anywhere by the gate itself.

---

## Compatibility Matrix

| Feature | Minimum Requirement |
|---|---|
| git | 2.20 (for `--diff-filter=ACMR`) |
| macOS | 11.0 |
| Linux kernel | 3.2 (musl static binary) |
| Windows | 10 / Server 2016 |

The gate works in:

- Shallow clones (`--depth=1`)
- Git worktrees (`git worktree add`)
- Monorepos (config discovery walks up from cwd)
- Repositories with submodules (submodule contents are not staged to the parent repo)
