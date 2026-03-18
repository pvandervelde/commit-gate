# Edge Cases

## EC-01: No staged files

- Condition: `git diff --cached --name-only` returns empty
- Expected: Gate exits 0 with a message "No staged files. Nothing to check."
- Rationale: An empty commit is not the gate's problem to solve. Git itself may reject it depending on config.

## EC-02: Binary files staged

- Condition: Staged files include binary files (images, compiled objects)
- Expected: Binary files are included in the staged file list for path checks but excluded from content-based checks (stub scan). Command checks receive the full file list — the external tool decides how to handle binary files.
- Rationale: Path restrictions apply to all files. Content scanning of binary files would produce garbage matches.

## EC-03: Staged file deleted

- Condition: A file is staged for deletion (`git rm`)
- Expected: Deleted files are excluded from the staged file list. The `--diff-filter=ACMR` filter already handles this (A=added, C=copied, M=modified, R=renamed — D=deleted is excluded).
- Rationale: You cannot check content that no longer exists. Deletion is always allowed.

## EC-04: File path contains spaces or special characters

- Condition: Staged file path is `src/my module/auth service.rs`
- Expected: Path is correctly shell-escaped in `{files}` substitution. Path check handles it correctly.
- Rationale: While unusual, spaces in paths are valid. Shell escaping failures would silently corrupt the command.

## EC-05: Very large staged file

- Condition: A staged file is 50MB (e.g. a generated file)
- Expected: Content-based checks (stub scan) read the full file. No memory limit is enforced by the gate — the operating system's virtual memory handles it.
- Rationale: The gate should not impose arbitrary size limits. If large files are a problem, address it with a dedicated check (e.g. `command = "find {files} -size +1M"`) or a `.gitattributes` rule.

## EC-06: Command check produces no output

- Condition: A command check exits with non-zero but writes nothing to stdout or stderr
- Expected: The check fails with an empty detail field. The remediation from the config is still present.
- Rationale: Some tools exit non-zero without explanation. The remediation field ensures the developer has something actionable.

## EC-07: Command check writes to stderr only

- Condition: A command check writes diagnostics to stderr, nothing to stdout
- Expected: stderr content is included in the diagnostic. The gate merges stdout and stderr in the check result.
- Rationale: Many tools (compilers, linters) write to stderr. Ignoring it would lose critical diagnostic information.

## EC-08: Concurrent gate invocations

- Condition: Two git commits in parallel (e.g. two worktrees, or a race condition)
- Expected: Each invocation operates independently. The gate holds no locks, writes no state. Git's own locking handles concurrent commit attempts.
- Rationale: The gate is a read-only observer of the staging area. Concurrent invocations cannot interfere.

## EC-09: Config file changes are staged

- Condition: `.commitgate.toml` itself is in the staged file list
- Expected: The gate reads the config from the filesystem (working tree), not from the git staging area. Config changes take effect on the next commit, not the current one.
- Rationale: The config must be stable during the check pipeline. Reading the staged version could produce paradoxes (e.g. a config that removes a check mid-pipeline). The working-tree file is the developer's current intent, and it is safe to assume it has not been broken between `git add` and the commit.

## EC-10: Check command modifies staged files

- Condition: A command check (e.g. a formatter with `--write` instead of `--check`) modifies files
- Expected: The gate does not detect or prevent this. The working tree may change, but the staging area is unaffected (the gate reads from the index). The commit will contain the original staged content.
- Rationale: The gate operates on the staging area snapshot taken at the start. Mutable commands are a configuration error, not a gate failure. Documentation should warn against using `--write` / `--fix` flags in check commands.

## EC-11: COMMITGATE_MODE set but no [modes] section in config

- Condition: Environment variable set but config has no mode definitions
- Expected: Gate exits with code 2 and a message explaining that the mode was requested but no modes are configured.
- Rationale: This is a configuration mismatch, not a silent no-op.

## EC-12: Check command contains {files} but no files match its extension

- Condition: `command = "cargo clippy"` configured, but only `.md` files are staged
- Expected: The command runs anyway with the full command string (no `{files}` substitution needed since the command doesn't use it, or empty file list if it does). The external tool decides if zero matching files is a pass or fail.
- Rationale: The gate does not know which extensions each command cares about. Extension filtering is the check command's responsibility.

## EC-13: Git not installed

- Condition: `git` binary not found in PATH
- Expected: Gate exits with code 2 and a message: "git not found in PATH. Commit Gate requires git."
- Rationale: Without git, the gate cannot read staged files or determine the repository root.

## EC-14: Not inside a git repository

- Condition: Gate invoked from a directory that is not inside a git repo
- Expected: Gate exits with code 2 with a clear message about not being in a git repository.
- Rationale: The gate is fundamentally a git tool. Operating outside git is meaningless.

## EC-15: Symlinks in staged files

- Condition: A staged file is a symlink
- Expected: The symlink itself is checked (path and target), not the target file's content. `git show :path` for a symlink returns the symlink target path.
- Rationale: Following symlinks during content scanning could read files outside the repository or create infinite loops.

## EC-16: UTF-8 encoding issues in staged files

- Condition: A staged `.rs` file contains invalid UTF-8 bytes
- Expected: The stub check treats the file as a byte sequence and performs byte-level pattern matching. It does not crash on invalid UTF-8. Line numbers may be approximate.
- Rationale: Rust source files should be UTF-8, but the gate should not crash on malformed files. Let the compiler report the encoding error.

## EC-17: Renamed files in `{files}` substitution

- Condition: A file is staged as a rename (`git mv old.rs new.rs`)
- Expected: The `--diff-filter=ACMR` output for renames uses the new path only (e.g. `new.rs`, not `old.rs -> new.rs`). The `{files}` substitution contains the new path. The stub check reads staged content using the new path. Command checks receive only the new path.
- Rationale: Git's `--name-only` output for renames produces the new path when `--diff-filter=R` is included, regardless of rename detection format. The old path no longer exists in the staging area and is irrelevant to checks.

## EC-18: Windows path separators in path check

- Condition: Gate runs on Windows against a mode with `writable = ["src"]`, and a staged file path is returned by git as `src/auth/service.rs` (forward slash)
- Expected: The path check performs prefix matching using forward-slash-normalised paths. Git always returns paths with forward slashes, even on Windows. The gate never converts to backslashes internally. The path `src/auth/service.rs` correctly matches the prefix `src`.
- Rationale: `git diff --cached --name-only` uses forward slashes on all platforms. Normalising to forward slashes is simpler and more portable than bifurcating on OS.

## EC-19: Mode active but `paths` check not in `sequence`

- Condition: `COMMITGATE_MODE=tester` is set, but `sequence` does not include `paths`
- Expected: The gate does NOT automatically prepend a path check. Path enforcement only occurs when `paths` is explicitly listed in `sequence`. If the user omits it, path restrictions are silently not enforced.
- Note: This is a potential misconfiguration. The gate SHOULD emit a warning to stderr when a mode is active but `paths` is absent from `sequence`, so the user knows enforcement is disabled. The warning does not change the exit code.
- Rationale: Silently injecting behaviour not requested by the config violates the principle that `sequence` is the single source of truth for execution order.

## EC-20: `{files}` substitution with an empty string

- Condition: `fail_fast = false` is set. One or more files are staged, but after extension filtering a command check like `prettier --check {files}` has zero matching files. Or `{files}` appears in a command but no files are staged at all.
- Expected: When `{files}` expands to an empty string the gate substitutes an empty string into the command, which becomes e.g. `prettier --check`. The external tool is responsible for handling empty input. The gate does NOT skip command checks solely because the file list is empty (only CG-PIPE-05 applies when there are zero staged files globally).
- Rationale: Extension-aware filtering is the external tool's responsibility. The gate cannot know which extensions each tool cares about. Skipping the command silently would hide misconfiguration.

## EC-21: `sequence` references an undefined check name

- Condition: `sequence = ["formatting", "nonexistent"]` where there is no `[checks.nonexistent]` section in the config
- Expected: Gate exits with code 2 (gate error) before any checks run. The error message names the undefined check and its position in the sequence.
- Rationale: An undefined check name in `sequence` is a configuration error, not a silent no-op. Proceeding would leave the developer believing the check ran when it did not. This is similar to CG-CFG-04 (unknown type) but for unknown names.
