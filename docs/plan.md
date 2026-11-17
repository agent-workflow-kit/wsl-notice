# wsl-notice: implementation plan

Implements [interface.md](interface.md) and [design.md](design.md). The issues for each stage are expanded into bite-sized test-first steps when they are picked up; this plan fixes the order, the conventions and the module map.

## Conventions for every stage

- **Gateway boundary.** All process starts and clock access go through `src/wsl_notice/gateway.py`; modules never import `subprocess` or `time` directly. Tests substitute fakes.
- **Fakes.** A fake `powershell.exe` in `tests/fakebin/` records its argv, stdin and environment and returns each row of the result table: exit 0, exit 3 with each stage token, exit 1 with `UtilAcceptVsock` once and twice, exit 7, a hang that ignores TERM, stdin closed early and a missing binary. There are no sibling tools to pin.
- **Process and live split.** Process tests run the CLI as a subprocess with the fake given as an absolute `powershell_bin` in the test config (there is no `PATH` search); WSL detection and the interop sockets are faked through the gateway. `live` tests (pytest marker, never in CI, WSL only) run real PowerShell with the spike's injection fixtures and need a person to confirm display; they are listed in the live-verification issue.
- **Golden files.** `--json` outputs, the stdin line and log records have golden files; the constant PowerShell script has a golden test; schemas have positive and negative samples.
- **Spike regressions.** Every behavior in design.md's live-verification table becomes a fake test or a `live` test.
- **No text in argv or environment.** A test passes marker strings (quotes, `$`, backticks, newlines, emoji, `<`, `&`) and asserts none appears in the fake's recorded argv or environment.
- **`make verify`** passes before every commit; each paragraph in markdown is one line.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `gateway.py` | 0 | Process start, clock. |
| `cli.py` | 0, 3, 4 | Parsing, dispatch, exit codes, output. |
| `payload.py` | 1 | Input parsing, validation, sanitizer. |
| `render.py` | 1 | Title and body assembly, truncation, the stdin JSON line. |
| `config.py` | 2 | Config, drop-ins, durations (no environment overrides). |
| `log.py` | 2 | Log records and rotation. |
| `platform.py` | 3 | WSL detection, interop socket check, child environment. |
| `toast.py` | 3 | Constant script, process start, result mapping, the single retry. |

## Stage 0: Scaffold, rename, gateway and display gate

**Goal**: The renamed package installs, prints `--version`, and has the gateway and the fake PowerShell in place; a person has seen a toast under the default identity.
**Success Criteria**: `make verify` green; no occurrence of the old name remains; schemas load; **0.1.0 display gate**: a person confirms a toast on screen under the default identity (if not, find a registered identity or document that an `app_id` must be set, before 0.1.0 is released). The constant script of [Constant script](interface.md#constant-script) is run once for real.
**Tests**: version output golden (`wsl-notice <version>`); fake PowerShell contract tests; schema positive and negative samples.
**Status**: Not Started

## Stage 1: Input, sanitizer and rendering

**Goal**: Validate and render a notification without starting any process.
**Success Criteria**: `send --dry-run` prints the exact JSON line for stdin, file and flag input with limits and the sanitizer applied.
**Tests**: sanitizer table (controls including NUL, bidi, ZWSP, CRLF, lone surrogates, non-BMP emoji, and the rows listed in design.md: U+061C, U+2028, U+2029, U+0085, U+E0041, U+00AD, U+FE0F, U+FFFE, U+FFFF, U+FDD0, a U+3164-only message, `a\n \n \nb`, a title with LF and U+2028, a fullwidth space only); input table (empty stdin, `[]`, `null`, `"x"`, BOM, duplicate keys, `NaN`, 100000-deep nesting, 1 MiB + 1 bytes, a FIFO or `/dev/zero` as `--file`, a directory, a symlink to a regular file accepted); link table (`HTTPS://x.example` accepted and rendered lowercase, trailing LF, `https://a.example\@evil.example`, `https://:80/`, `http://[::1]/`, `javascript:x` as `link_scheme`, `example.com:80/x` as `link_scheme`, non-ASCII, 1025 characters, `link_hosts` suffix match); title and body cut table with exact outputs; every refusal reason; flag and stdin parity; golden hashes and serialization.
**Status**: Not Started

## Stage 2: Config, state directory and log

**Goal**: Config loading with validation (PyYAML and Pydantic added here) and the state directory and log.
**Success Criteria**: invalid config exits 2 naming the key; log records match the schema and never contain text; an unusable state directory never changes a send's exit code.
**Tests**: config precedence (flags, config, defaults; no environment overrides); duration table (`10`, `"10s"`, `"1m"`, `"500ms"` out of range, `"1h"` refused, `.nan`, `true`, `"1e3"` refused, `-1`); `app_id`, `powershell_bin` and `link_hosts` grammar; unusable state directory (symlink, mode 0755, read-only, wrong owner) gives one warning; `HOME` unset; relative `XDG_STATE_HOME` ignored; log golden files, concurrent rotation with two processes.
**Status**: Not Started

## Stage 3: Toast

**Goal**: A real `send` through the fake PowerShell with the full result table.
**Success Criteria**: the argument list is identical for any notification; every result row maps to its exit code and envelope; outside WSL the tool exits 12 `not_wsl` (a dry run still works); the deadline stops only the exact child and its group (SIGTERM, 2 s, SIGKILL); the child gets only the allowlisted environment and working directory `/`.
**Tests**: result table (fake powershell per row); the single `UtilAcceptVsock` retry once and twice; timeout gives `outcome_unknown` with no retry (fake that ignores TERM; SIGTERM to wsl-notice does the same); BrokenPipe on stdin; missing binary gives `tool_missing` with install text; `not_wsl` through the gateway; stale interop socket test with a fake `/run/WSL` directory; child environment and working directory assertions; constant-script golden and the injection-XML golden; no-text-in-argv test; envelope test per code; `enabled: false` exits 22; usage error with `--json`.
**Status**: Not Started

## Stage 4: `validate`, live verification and docs

**Goal**: `validate [--probe] [--test-message]`, the live suite and user documentation.
**Success Criteria**: probe reports the PowerShell version and start time; the live suite settles the still-to-verify items (display under the default identity, truncation, Focus Assist); README documents requirements and troubleshooting.
**Tests**: validate golden files (it never prompts; `--test-message` sends; disabled gives exit 22); live tests (plain toast, injection fixtures, link tap on an https URL, a long body, non-ASCII, an empty message refusal).
**Status**: Not Started

## Version roadmap

- **0.x**: Stages 0 to 4.
- **0.1.0 gate**: a person has seen a toast under the default identity (stage 0); until then the docs do not call the tool working.
- **1.0** when the live suite has passed and the tool has run in daily use through a routing tool for two weeks without an interface change.
- **1.x, additive**: replacement by tag and group through the dedupe key, a registered application identity with an icon, scenario and sound by level.
- **2.0**: nothing planned. Notifiers for native Windows, other Linux desktops and macOS are separate tools, not versions of this one.
