# Responsibilities

## ConfigLoader

**Responsibilities:**
- Knows: the config file discovery algorithm (walk up from cwd)
- Knows: the TOML schema for `.commitgate.toml`
- Knows: default values for all optional fields
- Does: discovers and parses the configuration file
- Does: validates the configuration and produces human-readable errors on invalid config
- Does: merges defaults with user-specified values

**Collaborators:**
- Pipeline (provides the parsed configuration)

**Roles:**
- Parser: translates TOML into internal configuration types
- Validator: rejects invalid configurations before any checks run

---

## Pipeline

**Responsibilities:**
- Knows: the ordered sequence of checks to execute
- Knows: whether to fail-fast or run all checks
- Knows: the active mode (if any) and its path restrictions
- Does: orchestrates check execution in sequence
- Does: collects results from each check into an aggregate gate result
- Does: halts on first failure when fail-fast is enabled
- Does: enforces per-check timeouts

**Collaborators:**
- ConfigLoader (provides check sequence and mode)
- Check implementations (delegates validation to each check)
- StagedFiles (provides the list of files to validate)
- OutputFormatter (passes gate result for rendering)

**Roles:**
- Orchestrator: coordinates the check execution sequence
- Aggregator: combines individual check results into a gate result

---

## StagedFiles

**Responsibilities:**
- Knows: how to query git for the list of staged files
- Knows: how to read staged file content (via `git show :path`)
- Does: provides the list of staged file paths
- Does: provides staged file content for pattern scanning
- Does: filters the file list by extension when requested

**Collaborators:**
- Pipeline (provides staged file list and content)
- PathCheck (provides file paths for validation)
- StubCheck (provides file content for scanning)

**Roles:**
- Data source: abstracts git staging area access

---

## PathCheck (built-in)

**Responsibilities:**
- Knows: the writable paths for the active mode
- Knows: globally allowed paths (docs/, .llm/, etc.)
- Does: validates each staged file against the allowed path set
- Does: reports which files violated which path restrictions

**Collaborators:**
- StagedFiles (provides paths to check)
- Pipeline (receives check result)

**Roles:**
- Validator: enforces path restrictions per mode

---

## StubCheck (built-in)

**Responsibilities:**
- Knows: the list of stub patterns to scan for (from config)
- Knows: which file extensions to scan (from config)
- Does: scans staged file content for stub patterns
- Does: reports file path, line number, and matched pattern for each hit

**Collaborators:**
- StagedFiles (provides file content to scan)
- Pipeline (receives check result)

**Roles:**
- Scanner: detects incomplete implementations

---

## MessageCheck (built-in)

**Responsibilities:**
- Knows: commit message validation rules (min length, format, forbidden patterns)
- Does: validates a commit message against configured rules
- Does: reports which specific rule was violated

**Collaborators:**
- Pipeline (receives check result)

**Roles:**
- Validator: enforces commit message quality

---

## CommandCheck

**Responsibilities:**
- Knows: the shell command template to execute
- Knows: how to substitute `{files}` with the staged file list
- Knows: the per-check timeout
- Does: executes the command as a subprocess
- Does: captures stdout and stderr
- Does: interprets exit code (0 = pass, non-zero = fail)
- Does: truncates output to the configured line limit
- Does: kills the subprocess on timeout

**Collaborators:**
- StagedFiles (provides file paths for `{files}` substitution)
- Pipeline (receives check result)

**Roles:**
- Executor: runs external tools and translates their output into check results

---

## OutputFormatter

**Responsibilities:**
- Knows: whether to produce human-readable or JSON output
- Knows: the terminal width and colour support (for human output)
- Does: renders a gate result as coloured terminal text (human mode)
- Does: renders a gate result as a single JSON object (JSON mode)
- Does: writes output to stdout only (never stderr for structured output)

**Collaborators:**
- Pipeline (provides gate result to render)

**Roles:**
- Presenter: translates internal results into external representations

---

## CLI

**Responsibilities:**
- Knows: the command-line interface (subcommands, flags, env vars)
- Does: parses command-line arguments
- Does: determines the operating mode (pre-commit hook vs standalone check)
- Does: reads `COMMITGATE_MODE` from environment
- Does: sets the process exit code (0 = pass or success, 1 = check failure, 2 = gate error)

**Collaborators:**
- ConfigLoader (triggers config discovery)
- Pipeline (triggers check execution)
- OutputFormatter (triggers output rendering)

**Roles:**
- Entry point: wires all components together and manages the process lifecycle

---

## McpServer

**Responsibilities:**
- Knows: the MCP stdio protocol (JSON-RPC framing, tool registration)
- Knows: the `propose_commit` tool schema (accepted inputs, returned shape)
- Does: starts a stdio MCP server loop that accepts tool-call requests
- Does: calls `commitgate check --staged` logic (via Pipeline) on `propose_commit` invocation
- Does: executes the actual git commit (`git commit -m <message>`) if the pipeline passes
- Does: returns the full `GateResult` as the tool response (success or failure)
- Does: returns a structured error if the pipeline cannot run (gate error)

**Collaborators:**
- ConfigLoader (provides configuration)
- Pipeline (runs the check pipeline)
- GitCli (executes the commit on success)

**Roles:**
- Transport adapter: translates MCP tool-call protocol into pipeline execution and back
