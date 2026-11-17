# wsl-notice: implementation plan

Implements [interface.md](interface.md) and [design.md](design.md). The issues for each stage are expanded into bite-sized test-first steps when they are picked up; this plan fixes the order, the conventions and the module map.

## Conventions for every stage

- **Gateway boundary.** All process starts, input reads, the home directory, the uid and clock access go through `src/wsl_notice/gateway.py`; modules never import `subprocess` or `time` directly. Tests substitute fakes.
- **Fakes.** A fake `powershell.exe` in `tests/fakebin/` records its argv, stdin and environment and returns each row of the result table: exit 0, exit 3 with each stage token, exit 1 with `UtilAcceptVsock` once and twice, exit 7, a hang that ignores TERM, stdin closed early and a missing binary. There are no sibling tools to pin.
- **Process and live split.** Process tests call `cli.main(argv, env, gateway)` in-process with a fake gateway (process runner, `/proc/sys/kernel/osrelease` text, `/run/WSL` listing and sockets, clock, home directory and uid, input reading, state-directory hooks); no test-only environment seam is allowed. A small set of real-subprocess tests runs `tests/fakebin/powershell.exe` as an absolute `powershell_bin` (there is no `PATH` search) for the stdin pipe, process-group signals and BrokenPipe. `live` tests (pytest marker, never in CI, WSL only) run real PowerShell with the spike's injection fixtures and need a person to confirm display; they are listed in the live-verification issue.
- **Golden files.** `--json` outputs, the stdin line and log records have golden files; the constant PowerShell script has a golden test (the listing contains no non-ASCII byte, and `\uFFFE\uFFFF` appear as escapes; the encoded argument is the base64 of the UTF-16LE text with LF line ends; one case pins that `\/Date(...)\/` cannot be produced); schemas have positive and negative samples, including a value ending in LF for every pattern.
- **Spike regressions.** Every behavior in design.md's live-verification table becomes a fake test or a `live` test.
- **No text in argv or environment.** A test passes marker strings (quotes, `$`, backticks, newlines, emoji, `<`, `&`) and asserts none appears in the fake's recorded argv or environment.
- **`make verify`** passes before every commit; each paragraph in markdown is one line.

## Module map

| Module | Stage | Responsibility |
|---|---|---|
| `gateway.py` | 0 | Process start, clock, `read_source(fd_or_path, max_bytes, remaining)`, `home_dir()`, `uid()`, the interop sockets. |
| `cli.py` | 0, 3, 4 | Parsing, dispatch, exit codes, output. |
| `config.py` | 1 | Config from the password-database home, drop-ins, strict YAML loader, durations (no environment overrides). |
| `log.py` | 1 | Log records, bounded lock, rotation. |
| `payload.py` | 2 | Input parsing, validation, sanitizer. |
| `render.py` | 2 | Title and body assembly, truncation, the stdin JSON line. |
| `host.py` | 3 | WSL detection, interop socket choice, child environment. |
| `toast.py` | 3 | Constant script, process start, result mapping, the single retry. |

## Stage 0: Scaffold, rename, gateway and display gate

**Goal**: The renamed package installs, prints `--version`, and has the gateway and the fake PowerShell in place; a person has seen a toast under the default identity.
**Success Criteria**: `make verify` green; no occurrence of the old name remains; schemas load; **0.1.0 display gate**: a person confirms a toast on screen under the default identity (if not, find a registered identity or document that an `app_id` must be set, before 0.1.0 is released). The first task is to run the listing of [Constant script](interface.md#constant-script) verbatim once for real (`[Console]::In.ReadLine()` under `-EncodedCommand -NonInteractive` with redirected stdin) and record the result in design.md before stage 3; the display gate stays.
**Tests**: version output golden (`wsl-notice <version>`); fake PowerShell contract tests; schema positive and negative samples.
**Status**: Not Started

## Stage 1: Config, state directory and log

**Goal**: Config loading with validation (PyYAML and Pydantic are added here) and the state directory and log.
**Success Criteria**: invalid config exits 2 naming the key; log records match the schema and never contain text; an unusable state directory never changes a send's exit code.
**Tests**: config precedence (flags, config, defaults; no environment overrides); `HOME` and `XDG_CONFIG_HOME` set to a directory with a hostile config (ignored), `HOME` unset, `--config` plus drop-ins in its directory; a strict-loader table (`enabled: no`, `off`, `yes`, `timeout: 1:30`, `0x10`, `1_0`, `.nan` all `config_invalid`; `enabled: true`, `timeout: 10`, `timeout: 1.5` accepted); `[` repeated 1000 times and a 413-byte alias bomb (`config_invalid` or fast, no hang); a YAML error whose text would quote a fake secret (absent from stdout, stderr and the envelope); an unknown key of 5000 characters (`details.key` `?`); `level_prefix: "A\nB"` and `"INFO  x"` (two spaces) refused; duration table (`10`, `"10s"`, `"1m"`, `"500ms"` out of range, `"1h"` refused, `.nan`, `true`, `"1e3"` refused, `-1`); `app_id`, `powershell_bin` and `link_hosts` grammar (entries `com` and `.example.com` are `config_invalid`); `config.example.yaml` validates against `schemas/config.json` and loads with the stated defaults; schema samples with a trailing LF per pattern; unusable state directory (symlink, mode 0755, read-only, wrong owner, a path where `os.open` raises) gives one warning; a FIFO named `log.jsonl` and `log.lock` (send returns at once, one warning, exit code unchanged); a `flock` held by another process (record skipped after about 1 s, exit code unchanged); a symlinked `log.lock`; rotation reopen (the record lands in the new `log.jsonl`, not `.1`); relative `XDG_STATE_HOME` ignored; log golden files (`ts` in UTC with milliseconds and `Z`), concurrent rotation with two processes.
**Status**: Not Started

## Stage 2: Input, sanitizer and rendering

**Goal**: Validate and render a notification without starting any process. Needs stage 1 for the config.
**Success Criteria**: `send --dry-run` prints the exact JSON line for stdin, file and flag input with limits and the sanitizer applied.
**Tests**: sanitizer table (controls including NUL, bidi, ZWSP, CRLF, lone surrogates, non-BMP emoji, and the rows listed in design.md: U+061C, U+2028, U+2029, U+0085, U+E0041, U+00AD, U+FE0F, U+FFFE, U+FFFF, U+FDD0, a U+3164-only message, `a\n \n \nb`, a title with LF and U+2028, a fullwidth space only; U+180F, U+2065, U+FFF0, U+E0000, U+E0002, U+E01F0, U+E0FFF and U+16FE4 removed; a message of only U+2D7F, only U+1107F or only U+0301 is `empty_message`, while the letter a followed by U+0301 is not empty; a test that enumerates `unicodedata` for the default-ignorable ranges listed in design.md and asserts none survives); input table (empty stdin, `[]`, `null`, `"x"`, `{}` and `{"title":"x"}` as `empty_message`, BOM, duplicate keys, `NaN`, 100000-deep nesting, a 5000-digit integer, `{"message":"a","<5000-char key>":1}` (pointer `/?`), 1 MiB + 1 bytes, a regular file larger than the cap, a FIFO or `/dev/zero` as `--file`, a directory, a symlink to a regular file accepted; stdin that sends 10 bytes and stalls (exit 20 at the deadline, no hang); `--stdin-json` on a pseudo-terminal (exit 2); closed stdin `<&-`; SIGTERM during input (exit 20, pointer `""`); `--stdin-json --file x --dedupe-key=...` with `enabled: false` (exit 2, not 22); a disabled tool with a bad JSON source combination (exit 2)); link table (`HTTPS://x.example` accepted and rendered lowercase, trailing LF, `https://a.example\@evil.example`, `https://:80/`, `http://[::1]/`, `javascript:x` as `link_scheme`, `example.com:80/x` as `link_scheme`, non-ASCII, 1025 characters, `link_hosts` suffix match; refused: `https://a.example/"x`, `https://a.example/a|b`, `https://a.example/{x}`, `https://a.example/%zz`, `https://a.example#a#b`, `https://[8]/`, `https://[a]/`, `https://[:]#x`, an IPv6 literal with `link_hosts` set; accepted: `https://a.example/%41`, `https://a.example/p'q(r)`, `https://[::1]/`, `https://2130706433/` with `link_hosts` unset); title and body cut table with exact outputs; every refusal reason; flag and stdin parity; parser table (`--conf` and `--stdin` rejected, `--config=x` accepted, `--dedupe-key=-abc` accepted, `--dedupe-key -abc` exit 2, `--config x send` and `send --config y` both honoured: a global flag given before the subcommand is not overwritten); golden hashes and serialization.
**Status**: Not Started

## Stage 3: Toast

**Goal**: A real `send` through the fake PowerShell with the full result table.
**Success Criteria**: the argument list is identical for any notification; every result row maps to its exit code and envelope; outside WSL the tool exits 12 `not_wsl` (a dry run still works); the deadline stops only the exact child and its group (SIGTERM, 2 s, SIGKILL); the child gets only the allowlisted environment and working directory `/`.
**Tests**: result table (fake powershell per row); the single `UtilAcceptVsock` retry once and twice; result table rows: fake exits 7, exit 1 without the substring, a kill by signal (-9, -11), exit 3 with each token, exit 3 with other stderr (`unknown`), a token with exit 1 (`outcome_unknown`), `UtilAcceptVsock` once (sent, attempts 2), twice (`interop_failed`, attempts 2), once with `timeout: 5s` and a first attempt that takes 3 s (`interop_failed`, attempts 1, no retry), a deadline before the spawn (`interop_failed`, attempts null), a deadline after it (`outcome_unknown`, `exit_status` null); stdout and stderr that fill their pipes (no deadlock, 4 KiB caps); a fake runner that ignores TERM (assert SIGKILL and total wall time at most timeout + 4 s + 1 s); a fake whose `poll()` never reports an exit after KILL (the result still stands); SIGKILL not sent after the child was reaped by the TERM (assert no `killpg` call); SIGHUP, a second SIGINT during the stop sequence, SIGTERM during input; BrokenPipe on write, close and flush; missing binary gives `tool_missing` with install text; `not_wsl` through the gateway; interop socket tests with a fake `/run/WSL` through the gateway: stale inherited (fallback chosen, passed to the child only), inherited root-owned that did not accept and no fallback (passed anyway), inherited not root-owned (not passed; the planted listener test: no connection reaches it), six candidates (five tried), a symlink skipped, equal mtimes (stable order by name), a connect that stalls (200 ms, within the deadline), the retry re-runs the choice; child environment and working directory assertions; constant-script golden and the injection-XML golden; no-text-in-argv test; envelope test per code; `enabled: false` exits 22; usage error with `--json`.
**Status**: Not Started

## Stage 4: `validate`, live verification and docs

**Goal**: `validate [--probe] [--test-message]`, the live suite and user documentation.
**Success Criteria**: probe reports the PowerShell version and start time; the live suite settles the still-to-verify items (display under the default identity, truncation, Focus Assist); README documents requirements and troubleshooting.
**Tests**: validate golden files (it never prompts; `--test-message` sends; `validate --dry-run --test-message` still sends); disabled rows: plain `validate` exits 0 with `enabled: false`, a broken `powershell_bin` of a disabled tool exits 12, `--test-message` and `--probe` of a disabled tool exit 22 with no call and before any check; a `{"outcome":"disabled"}` sample against `validate.json`; probe output matching and failing (`[0-9]+(\.[0-9]+){1,3}`, interop failure, deadline); live tests (plain toast, injection fixtures, link tap on an https URL, a long body, non-ASCII; with `WSL_INTEROP` alone and a connect test that does not disturb a later start). The empty-message refusal needs no PowerShell and is tested in stage 2.
**Status**: Not Started

## Version roadmap

- **0.x**: Stages 0 to 4.
- **0.1.0 gate**: a person has seen a toast under the default identity (stage 0); until then the docs do not call the tool working.
- **1.0** when the live suite has passed and the tool has run in daily use through a routing tool for two weeks without an interface change.
- **1.x, additive**: replacement by tag and group through the dedupe key, a registered application identity with an icon, scenario and sound by level.
- **2.0**: nothing planned. Notifiers for native Windows, other Linux desktops and macOS are separate tools, not versions of this one.
