# Tradeoffs

## T-01: Shell command checks vs embedded tool support

**Decision:** Command checks invoke external tools via shell.

**Alternative considered:** Embed formatting/linting logic directly (e.g. link `rustfmt` as a library).

**Pros of shell commands:**

- Language-agnostic — works with any tool that has a CLI
- No recompilation when adding new check types
- Users already know how to invoke their tools
- Smaller binary size

**Cons of shell commands:**

- Subprocess overhead (typically 10-50ms per check)
- Shell escaping complexity for file paths with spaces/special characters
- Platform differences (`sh -c` vs `cmd /C`)
- Tool must be installed separately

**Rationale:** The gate is a coordinator, not a linter. Embedding tool logic would create a maintenance burden and limit the gate to specific languages. The subprocess overhead is negligible compared to the time spent running `cargo test`.

---

## T-02: Fail-fast vs run-all default

**Decision:** Fail-fast is the default.

**Alternative considered:** Run all checks and report all failures.

**Pros of fail-fast default:**

- Faster feedback cycle (don't wait for slow test suite when formatting is wrong)
- Less noise for agents (one problem at a time is easier to fix)
- Matches user expectation from CI pipelines (first error stops the build)

**Cons of fail-fast default:**

- Multiple issues require multiple fix-commit-retry cycles
- Developers may prefer to see all problems at once

**Rationale:** For agents, sequential correction is more reliable than parallel correction. For humans, `fail_fast = false` is one line of config away. The default optimises for the primary use case (quick feedback).

---

## T-03: Staged content vs working tree content

**Decision:** Always read from git staging area (`git show :path`).

**Alternative considered:** Read file content directly from the filesystem.

**Pros of staged content:**

- Validates exactly what will be committed
- Partial staging (`git add -p`) is handled correctly
- Unstaged changes don't contaminate the check

**Cons of staged content:**

- Slightly more complex implementation (shell out to git per file)
- Cannot detect issues in files that haven't been staged yet

**Rationale:** A commit gate must validate the commit, not the working tree. If the gate reads the filesystem, a developer could stage clean files but have dirty working tree content, and the gate would report false positives or negatives.

---

## T-04: Single binary vs plugin system for checks

**Decision:** Built-in checks + shell commands. No plugin system for v1.

**Alternative considered:** WASM plugin system for custom checks.

**Pros of no plugins:**

- Simpler architecture
- Faster startup (no plugin loading)
- Shell commands already provide extensibility

**Cons of no plugins:**

- Custom checks that need access to staged file content (like stub scanning with custom logic) must use shell scripts that call `git show`

**Rationale:** Shell commands cover the vast majority of check use cases. A plugin system adds significant complexity for a marginal benefit. If custom in-process checks become necessary, this can be added in v2.

---

## T-05: TOML vs YAML vs JSON for configuration

**Decision:** TOML.

**Alternative considered:** YAML, JSON.

**Pros of TOML:**

- Designed for configuration files (unlike JSON)
- Less ambiguous than YAML (no billion laughs, no boolean gotchas)
- Readable without a schema reference
- Good Rust ecosystem support (`toml` crate)
- Consistent with Rust tooling conventions (`Cargo.toml`)

**Cons of TOML:**

- Less familiar to developers from non-Rust ecosystems
- Nested structures can be verbose

**Rationale:** The target audience is Rust developers (initially). TOML is the natural choice. The config format is simple enough that TOML's nesting limitations don't matter.

---

## T-06: Exit code semantics (0/1 vs 0/1/2)

**Decision:** Three exit codes: 0 (pass), 1 (check failure), 2 (gate error).

**Alternative considered:** Two exit codes: 0 (pass), 1 (fail/error).

**Pros of three codes:**

- Scripts can distinguish "commit is bad" from "gate is broken"
- Agents can retry on gate errors but self-correct on check failures
- CI can alert differently on infrastructure issues vs code quality issues

**Cons of three codes:**

- Slightly more complex for simple scripts that just check `$?`

**Rationale:** The distinction between "your code has a problem" and "the tool has a problem" is fundamental. Conflating them leads to confusion in automated pipelines.

---

## T-07: Primary agent integration mechanism — git hook vs git-alias wrapper vs MCP/skills tool

**Decision:** Not fixed for v1. The integration mechanism is deployment-context dependent. All three approaches are valid; they are not mutually exclusive. Recommended default stack is documented below.

**Context:** Git hooks have three significant problems for AI agent use:

1. **Per-repo setup** — hooks must be configured in every repository; there is no automatic enforcement when an agent creates or clones a new repository
2. **Bypassable** — `git commit --no-verify` skips all hooks with no recourse
3. **Poor feedback channel** — hook output travels through git's hook pipe and is frequently lost or interleaved with git's own output; many agent implementations see only the combined exit code of `git commit`, not the gate's structured diagnostic

---

### Option A: Git Pre-Commit Hook (status quo)

**How it works:** `commitgate install` creates a `pre-commit` hook in the repository. The agent calls `git commit` normally; git invokes the hook.

**Pros:**

- Zero agent-side changes required
- Works for both human developers and agents with the same setup
- Familiar to anyone who has used git hooks

**Cons:**

- Per-repo: must be installed in every repository; ephemeral/cloned repos start without the gate
- Bypassable: `git commit --no-verify` skips it entirely
- Feedback loss: hook stdout/stderr may not reach the agent reliably; the agent sees "commit failed" with no structured reason
- "Randomly dies": from an agent's perspective, a hook failure looks identical to a git internal error — both produce a non-zero exit from `git commit` with unstructured text on stderr

**Best suited for:** Human developer workflows; repositories where the `.githooks/` directory is committed and `core.hooksPath` is set globally.

---

### Option B: Git Alias Wrapper

**How it works:** A wrapper script named `git` is placed first in `PATH` (typically via the container image or sandbox environment). The wrapper intercepts `git commit` and runs `commitgate check --staged --json` first. All other git subcommands are delegated directly to the real git binary.

```bash
#!/usr/bin/env bash
# /usr/local/bin/git  (wrapper, placed before real git in PATH)
# IMPORTANT: REAL_GIT must be an absolute path, never a PATH lookup.
# A PATH lookup would find this wrapper itself and recurse infinitely.
REAL_GIT=/usr/bin/git

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

**Pros:**

- Not bypassable: agent calls `git commit` as normal and has no visible mechanism to circumvent the gate
- Not per-repo: the wrapper is in the environment and applies to all repositories without any per-repo setup
- Feedback captured: the wrapper controls stdout/stderr and can cleanly separate gate output from git output
- Transparent: no agent-side changes required; the agent cannot distinguish the wrapper from real git

**Cons:**

- Requires controlling the agent's execution environment (container image, sandboxed PATH); not appropriate for human developer machines
- A wrapper bug is catastrophic: if the wrapper crashes, all git operations fail; rigorous testing and a clear fallback path are required
- Maintenance burden: the wrapper must stay in sync with git subcommand parsing if new interception points are needed
- Transparency is a double-edged sword: the agent cannot opt out even for intentional low-overhead commits (e.g. amending a commit after the gate has already run)
- Does not work for GUI git clients or tools that find git via absolute path

**Best suited for:** AI agent environments where the agent runs in a controlled container or sandbox; CI pipelines; multi-repo agent workspaces.

---

### Option C: MCP Tool / GitHub Copilot Skill

**How it works:** The gate exposes a structured commit tool (`propose_commit`) via a stdio MCP server (`commitgate mcp`) or a Copilot skill definition. The agent explicitly calls `propose_commit(files, message)` rather than `git commit`. The gate checks, then commits if everything passes, returning a structured result.

```
Agent → propose_commit({files: [...], message: "feat: ..."})
     ← { success: false, checks: [...], remediation: "..." }
Agent → (fixes code)
Agent → propose_commit({files: [...], message: "feat: ..."})
     ← { success: true }
```

**Pros:**

- Richest feedback: the agent receives the full `GateResult` as a typed tool return value — no output-capture problem
- Explicit: the agent is aware it is using a gated commit mechanism; integration intent is clear in the agent's tool calls
- Not per-repo: the MCP server runs in the agent's environment, not per-repository
- Self-correction loop is idiomatic: tool-use → failure → fix → tool-use again is the natural agent loop
- Works identically for Claude Code (MCP) and GitHub Copilot (Copilot Extension/Skill) with separate but equivalent definitions

**Cons:**

- Requires agent-side configuration: the MCP server or skill must be registered with the agent framework
- The MCP server process must be running during agent sessions; adds operational overhead
- Not applicable to agents that only support shell-command execution
- The agent can still call `git commit` directly if it is not constrained — Option C does not prevent bypass, it only provides an alternative path
- GitHub Copilot skill definition format and MCP server protocol may evolve; maintenance required

**Best suited for:** AI coding agents that support MCP or Copilot Extensions; scenarios where the richest possible feedback to the agent matters most.

---

### Recommended Deployment Strategy (v1)

| Deployment Context | Primary Integration | Bypass Risk | Feedback Quality |
|---|---|---|---|
| Human developer, local machine | Git hook (Option A) | Low (human error only) | Good (terminal TTY) |
| AI agent, containerised | Git alias wrapper (Option B) | None | Good |
| AI agent, MCP-capable (Claude Code) | MCP tool (Option C) | Low (agent must be constrained separately) | Excellent |
| AI agent, Copilot | Copilot Skill (Option C) | Low | Excellent |
| CI pipeline | Standalone CLI | N/A | Excellent (JSON output) |

**For maximum agent reliability:** deploy Option B (alias wrapper) in the container to prevent bypass, and Option C (MCP/skill) as the preferred commit path. The alias wrapper acts as a safety net if the agent ever falls back to raw `git commit`; the MCP tool gives the agent a proper feedback loop.
