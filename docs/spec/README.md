# Commit Gate — Specification

A configurable, fail-fast quality gate that validates proposed commits against a pipeline of checks before allowing them into git history.

## Purpose

Commit Gate is a standalone Rust binary that intercepts commit operations and runs a configurable sequence of quality checks. If all checks pass, the commit proceeds. If any check fails, the commit is rejected with structured, actionable feedback.

It is useful for both human developers and AI coding agents. For humans, it prevents accidental commits of incomplete or non-conforming code. For agents, it provides a machine-readable feedback loop that enables autonomous self-correction.

## Key Design Decisions

1. **Single binary, no runtime dependencies.** Ships as a static Rust executable. No Node.js, Python, or Docker required to run the gate itself (though the checks it invokes may require those).
2. **Configuration-driven.** All checks, patterns, and thresholds are defined in `.commitgate.toml` at the repo root. No behaviour is hardcoded.
3. **Dual output modes.** Human-readable coloured terminal output by default; structured JSON when invoked with `--json` or when stdout is not a TTY (for agent consumption).
4. **Fail-fast by default.** First failing check stops the pipeline. Configurable to run all checks and report all failures.
5. **Git-native integration.** Installs as a git pre-commit hook, but also works as a standalone CLI for pre-flight validation.
6. **Path-aware.** Optionally restricts which files are allowed in a commit based on a named mode (for agent role separation). This is the bridge to agentgate — the same config format, the same path rules.

## Specification Documents

| Document | Contents |
|---|---|
| [overview.md](overview.md) | System context, integration points, operational model |
| [vocabulary.md](vocabulary.md) | Domain concepts and their definitions |
| [responsibilities.md](responsibilities.md) | Component responsibilities (RDD) |
| [architecture.md](architecture.md) | Internal architecture and module boundaries |
| [assertions.md](assertions.md) | Behavioral assertions (testable specifications) |
| [constraints.md](constraints.md) | Implementation constraints and rules |
| [configuration.md](configuration.md) | Full TOML schema reference with example |
| [cli.md](cli.md) | CLI commands, flags, output formats |
| [tradeoffs.md](tradeoffs.md) | Design alternatives and decisions |
| [edge-cases.md](edge-cases.md) | Failure modes and non-standard flows |
| [security.md](security.md) | Security considerations |
| [testing.md](testing.md) | Testing strategy and coverage targets |
| [operations.md](operations.md) | Distribution, installation, CI integration, upgrading |

## Workflow

```
Architect (this spec)
    ↓
Interface Designer → concrete types, traits, CLI interface
    ↓
Planner → sequenced implementation tasks
    ↓
Tester → adversarial tests against the spec
    ↓
Coder → implementation
```

## Relationship to agentgate

Commit Gate is the **commit gateway** component extracted from the agentgate specification as a standalone tool. It can be used without Docker, without agent role separation, and without the launcher — it is purely the quality gate.

When agentgate is eventually built, it will use Commit Gate as its commit validation engine. The `.commitgate.toml` configuration is a subset of `.agentgate.toml` — the formats are compatible.
