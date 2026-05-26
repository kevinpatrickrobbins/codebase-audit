# Product-Type Sub-Reference: CLI Tool

Audit a command-line tool / utility binary.

---

## Detection

- `bin` field in `package.json` (Node-based CLI)
- `entry_points` console_scripts in `pyproject.toml` / `setup.py` (Python)
- `cmd/<name>/main.go` with `main()` function (Go)
- `Cargo.toml` `[[bin]]` (Rust)
- A primary `cli.{ts,js,py}` / `index.{ts,js,py}` entry without UI / server
  code
- Imports of `commander`, `yargs`, `clipanion`, `oclif` (Node), `click`,
  `typer`, `argparse` (Python), `cobra`, `urfave/cli` (Go), `clap` (Rust)

---

## Investigate

### Argument parsing

- **Library.** Modern Node CLIs typically use `commander`, `clipanion`,
  `cac`, or `oclif` (heavyweight). Manual `process.argv` parsing in non-trivial
  CLIs is a smell.
- **Subcommands.** For multi-command tools (`tool x`, `tool y`), is
  the structure consistent?
- **Help text.** `--help` complete and well-formatted? Each subcommand has
  its own help?
- **Version.** `--version` returns the package version (read from
  `package.json` / equivalent), not hardcoded.
- **Flags vs positional args** — conventional and consistent.
- **Short flags / long flags** mapped per convention (`-h` / `--help`,
  `-v` / `--verbose` or `--version`).

### Exit codes

- **0 = success, non-zero = error.** Hunt for `process.exit(0)` after a
  caught error or a thrown error not surfacing as non-zero — silently
  failing CLI is a common bug.
- **Specific exit codes** for distinct error classes (e.g., 64 misuse, 65
  data error, 66 no input — see sysexits.h conventions). Not always
  warranted, but matters for shell-script callers.

### TTY / piping behavior

- **Auto-detect TTY.** `process.stdout.isTTY` — colors, progress bars, and
  prompts on; raw output off when piped (`tool list | grep`).
- **`--no-color`** support (and respect `NO_COLOR` env var convention).
- **`--json` / `--format json`** for machine-readable output. Critical for
  any CLI meant to be scripted around. Stable JSON schema (versioned).
- **`--quiet` / `-q`** for log suppression.
- **`--verbose` / `-v`** with levels (`-v`, `-vv`, `-vvv`).

### Input

- **Stdin handling.** `cmd | tool` — accept stdin where natural.
- **Argument reading order.** Conventional: explicit args > stdin > config
  file > env var > default.

### Output

- **Stdout = data, stderr = noise.** Logs, progress bars, prompts to stderr;
  the actual command output to stdout. Mixing breaks `tool | jq`.
- **Color discipline** when piped (off by default).
- **Pretty vs raw** output modes.

### Errors

- **Helpful error messages** — point at the specific bad input.
- **Stack traces hidden** by default; `--debug` / `DEBUG=1` shows internals.
- **Exit on first vs continue.** For batch operations, `--fail-fast` vs
  collecting results.

### Configuration

- **Config file location.** Conventional: `~/.config/<tool>/config.json`
  (Linux), `~/Library/Application Support/<tool>/` (macOS), `%APPDATA%`
  (Windows). `cosmiconfig` / `conf` on Node; `confy` / similar elsewhere.
- **Env var fallback.** Ideally `TOOL_*` prefix.
- **Override hierarchy** documented.

### Distribution

- **npm install -g** — works everywhere Node is installed.
- **`npx tool`** — no install required (good DX for one-shot use).
- **Homebrew formula** for macOS.
- **Scoop / Winget** for Windows.
- **Pre-built binaries** for Go/Rust CLIs distributed via GitHub Releases —
  not requiring Node/Python toolchain.
- **Auto-update** for installed binaries (`update-notifier` on Node).
- **Reproducible install.** `engines` pin in package.json.

### Performance

- **Cold start.** A CLI that takes 2s to print `--help` is unusable for
  scripting. Hunt for top-level synchronous heavy imports.
- **Lazy-load subcommands.** Top-level imports only the dispatcher; the
  subcommand's deps load when that subcommand runs.
- **Bundling.** `pkg`, `nexe`, `bun build --compile` for distribution as a
  single binary.

### Updates & telemetry

- **Update notifier** — opt-out, respects offline.
- **Telemetry** — explicit opt-in or clearly disclosed; respects DO_NOT_TRACK.

### Security

- **Don't shell out unsafely.** `child_process.exec` with user input is a
  command injection vector.
- **API tokens** — read from env or platform keychain, not committed config.
- **Auto-update channel** signed if updates can replace the binary.

---

## Red flags

- Manual `process.argv` parsing in a non-trivial CLI.
- Errors caught and logged but `process.exit(0)` returned.
- Mixing logs and data on stdout — breaks pipe composition.
- Always-on color output even when piped.
- `--help` longer than 200 lines without subcommand structure.
- `tool --json` not supported despite the tool being scriptable.
- Cold start > 1s (top-level heavy imports).
- Single `if (subcommand === 'foo')` branching tree instead of a parser
  library.
- Hardcoded version string drifting from `package.json` / equivalent.
- `npm install -g` requires `sudo` (the binary's `bin` script lacks correct
  shebang or perms).

---

## Output section

```markdown
### CLI Tool

#### Argument parsing
- Library: <commander / yargs / oclif / cobra / clap / manual>
- Subcommand structure
- Help text quality (subcommand-level)
- --version source

#### Exit codes
- 0 / non-zero discipline
- Specific codes for error classes

#### TTY / piping
- TTY auto-detect
- NO_COLOR / --no-color
- --json / --format
- --quiet / --verbose

#### Output discipline
- stdout (data) vs stderr (noise) separation

#### Errors
- Specific messages
- Stack-trace hiding by default
- --debug / DEBUG=1

#### Configuration
- Config file location convention
- Env var fallback
- Override hierarchy

#### Distribution
- npm / npx / Homebrew / Scoop / Winget / pre-built binaries
- Auto-update mechanism

#### Performance
- Cold-start time
- Lazy-load subcommands
- Single-binary bundling

#### Security
- Shell-out hygiene
- Token storage
- Update channel signing

#### Findings
(table)
```
