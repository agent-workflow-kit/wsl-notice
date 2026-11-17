# wsl-notice: implementation plan

Implements [interface.md](interface.md) and [design.md](design.md). The issues for each stage are expanded into bite-sized test-first steps when they are picked up; this plan fixes the order, the conventions and the module map.

## Conventions for every stage

- **Gateway boundary.** All process starts and clock access go through `src/wsl_notice/gateway.py`; modules never import `subprocess` or `time` directly. Tests substitute fakes.
- **Fakes.** A fake `powershell.exe` in `tests/fakebin/` records its argv, stdin and environment and returns each row of the result table, including a hang, the `UtilAcceptVsock` failure and a missing binary. There are no sibling tools to pin.
- **Process and live split.** Process tests run the CLI as a subprocess with the fake on `PATH`. `live` tests (pytest marker, never in CI, WSL only) run real PowerShell with the spike's injection fixtures and need a person to confirm display; they are listed in the live-verification issue.
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
| `config.py` | 2 | Config, drop-ins, env overrides. |
| `log.py` | 2 | Log records and rotation. |
| `platform.py` | 3 | WSL detection and PowerShell resolution. |
| `toast.py` | 3 | Constant script, process start, result mapping, the single retry. |

## Stage 0: Scaffold, rename and gateway

**Goal**: The renamed package installs, prints `--version`, and has the gateway and the fake PowerShell in place.
**Success Criteria**: `make verify` green; no occurrence of the old name remains; schemas load.
**Tests**: version output golden (`wsl-notice <version>`); fake PowerShell contract tests; schema positive and negative samples.
**Status**: Not Started

## Stage 1: Input, sanitizer and rendering

**Goal**: Validate and render a notification without starting any process.
**Success Criteria**: `send --dry-run` prints the exact JSON line for stdin, file and flag input with limits and the sanitizer applied.
**Tests**: sanitizer table (controls including NUL, bidi, ZWSP, CRLF, lone surrogates, non-BMP emoji); title and body truncation at the boundary; every refusal reason; flag and stdin parity.
**Status**: Not Started

## Stage 2: Config, state directory and log

**Goal**: Config loading with validation and the safe state directory and log.
**Success Criteria**: invalid config exits 2 naming the key; log records match the schema and never contain text.
**Tests**: config precedence and env overrides; `app_id` and `powershell_bin` grammar; state directory checks; log golden files and rotation.
**Status**: Not Started

## Stage 3: Toast

**Goal**: A real `send` through the fake PowerShell with the full result table.
**Success Criteria**: the argument list is identical for any notification; every result row maps to its exit code and envelope; outside WSL the tool refuses; timeouts signal only the exact child and its group.
**Tests**: result table; the single `UtilAcceptVsock` retry; timeout gives `outcome_unknown` with no retry; missing binary gives `tool_missing` with install text; `not_wsl` refusal; constant-script golden; no-text-in-argv test; `enabled: false` exits 22.
**Status**: Not Started

## Stage 4: `validate`, live verification and docs

**Goal**: `validate [--probe] [--test-message]`, the live suite and user documentation.
**Success Criteria**: probe reports the PowerShell version and start time; the live suite settles the still-to-verify items (display under the default identity, truncation, Focus Assist); README documents requirements and troubleshooting.
**Tests**: validate golden files; confirmation on `/dev/tty` and `--yes`; live tests (plain toast, injection fixtures, link tap on an https URL, a long body, non-ASCII, an empty message refusal).
**Status**: Not Started

## Version roadmap

- **0.x**: Stages 0 to 4.
- **1.0** when the live suite has passed (including a person confirming that toasts appear) and the tool has run in daily use through a routing tool for two weeks without an interface change.
- **1.x, additive**: replacement by tag and group through the dedupe key, a registered application identity with an icon, scenario and sound by level.
- **2.0**: nothing planned. Notifiers for native Windows, other Linux desktops and macOS are separate tools, not versions of this one.
