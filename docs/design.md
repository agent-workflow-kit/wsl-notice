# wsl-notice: design

Status: 0.1.0-design, 2026-11-17. The rules are in [interface.md](interface.md), the single normative source; this document records decisions, modules and tests and links to interface headings without restating them. Architecture: `../awkit-docs/architecture.md` (v1.22).

## Role and layer

L1 primitive: show one Windows toast from a process running inside WSL. Works fully standalone. It is one of three sibling channel notifiers with the same contract (`slack-notice`, `ha-notice`), each designed to be called by a routing tool above it without knowing that tool exists. It was seeded as `desktop-notice`; the maintainer renamed it so the name says what it is, and future platform notifiers (native Windows, other Linux desktops, macOS) are separate tools: `win-notice`, `linux-notice`, `mac-notice`.

## Uses

| Dependency | Layer | Binding | Notes |
|---|---|---|---|
| Windows PowerShell 5.1 through WSL interop | L0 third party | none (not an edge of the tool graph) | One short-lived process per notification; see [Toast construction](interface.md#toast-construction). |
| Python standard library (`subprocess` through `gateway.py`) | L0 | none | No third-party runtime dependency beyond Pydantic and PyYAML. |
| Pydantic (config validation) | library | none | Required by the design guide for config. Pydantic errors are converted to messages built from `loc` and `type` only, because the default message contains the input value. The scaffold's `pyproject.toml` does not list it yet; adding it is part of stage 2 of [plan.md](plan.md). |
| PyYAML (config parsing) | library | none | The design guide mandates YAML config; only `yaml.safe_load` is used, on files of at most 64 KiB. Added with Pydantic in stage 2. The duration grammar is defined in [Config](interface.md#config) because the design guide has none. |

No tool dependencies: wsl-notice composes nothing.

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-1 | Stateless: no state database, no outbox, no retries after exit; one `log.jsonl`. See [Files](interface.md#files), [Log](interface.md#log). | Deduplication, quiet hours and history belong to the routing tool (maintainer decision, 2026-11-17). |
| D-2 | Common input contract: `title`, `message`, `level`, `link`; unknown fields refused; text from stdin, a file or flags, all sanitized alike. See [Notification](interface.md#notification). | Architecture 13.2 rule 2; short argv text is untrusted (maintainer decision, 2026-11-17). |
| D-3 | Named for what it is: `wsl-notice` (renamed from `desktop-notice`). | A tool tied to one platform's mechanism says so; siblings get their own names (maintainer decision, 2026-11-17). |
| D-4 | Notification text never touches argv or the environment: a constant `-EncodedCommand` script and one base64 JSON line on stdin. See [Toast construction](interface.md#toast-construction). | Architecture 13.2 rule 2; `powershell.exe`'s text would otherwise be visible in process lists on both sides (spike C checked `ps`, `/proc` and the Windows command line). |
| D-5 | The toast XML is built from text nodes (`InnerText`, `SetAttribute`), never by concatenation. | Naive concatenation let text inject images, looping audio and action buttons and broke on `<` and `&` (spike C). |
| D-6 | Base64 on stdin, not raw UTF-8. | The console input code page corrupted raw UTF-8 (spike C). |
| D-7 | `-EncodedCommand`, not `-Command -`. | `-Command -` runs stdin line by line, stops silently after the first statement of a multi-line block and gives no error exit (spike C). |
| D-8 | `link` becomes the toast's protocol-activation target, `http` and `https` only, printable ASCII, one regex, scheme compared case-insensitively and rendered lowercase; optional `link_hosts` restricts hosts and is unset by default. See [Notification](interface.md#notification). | The toast API accepts any scheme (`file:`, `ms-settings:` and custom protocols built without validation). The URL is hidden behind a tap, so printable ASCII removes bidi, zero-width and homograph display problems in one rule (maintainer decision, 2026-11-17). |
| D-9 | Output says `sent`, never `displayed`; exit 12 `toast_failed` only when PowerShell fails. | Display, dismissal and Focus Assist suppression are not observable from PowerShell 5.1 (spike C). |
| D-10 | One retry after 500 ms on the interop error `UtilAcceptVsock` (exit 1), inside the deadline and only when at least 2 s remain; before every spawn a stale `WSL_INTEROP` is replaced by the newest live `/run/WSL/*_interop` socket; a deadline after the spawn is `outcome_unknown` with no retry. See [Toast construction](interface.md#toast-construction). | The error appeared twice in the spike, right after a hung or killed call, so an immediate retry may fail again; a retry after a timeout could show two toasts. The stale-socket premise is unverified (V14). |
| D-11 | Limits sit below what Windows accepted (title 100, body 1000 by default). | 5000-character bodies were accepted but on-screen truncation is not observable. |
| D-12 | Default `app_id` is the Windows PowerShell identity; unregistered identities are allowed by config but warned about in the docs. | The PowerShell identity needs no registration or install; unregistered ones produced no error and possibly no toast. |
| D-13 | Exit codes and outcomes as in [Exit codes](interface.md#exit-codes); exit 22 for a disabled tool; `not_wsl` is exit 12 (retryable false). | Exit 20 means only that the payload was refused; environment faults are exit 12 codes, so a router has one rule: 20 drop and surface, 12 read the flags, 22 no-op (maintainer decision, 2026-11-17). Same as the sibling notifiers. |
| D-14 | No hooks in 0.x and no audit file; `config_invalid` is exit 2. See [Hooks](interface.md#hooks), [Log](interface.md#log). | As in the sibling notifiers. |
| D-15 | Sanitizer by Unicode category (Cc, Cf, Cs, Co) plus noncharacters and a short filler list, in a fixed order; Cn is not used. See [Sanitizer](interface.md#sanitizer). | Hand-listed ranges missed characters. Cn membership changes with the Unicode version of the running Python. The rule also removes the zero-width joiner and variation selector 16, so some emoji sequences render as separate or text-style emoji (accepted). U+FFFE and U+FFFF break `InnerText` (spike). |
| D-16 | `powershell_bin` is an absolute path, default `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe`; no `PATH` search; WSL detected only from `/proc/sys/kernel/osrelease`; `WSL_DISTRO_NAME` ignored. | Cron, systemd and a planted `powershell.exe` earlier in `PATH` become irrelevant; `WSL_DISTRO_NAME` is missing under cron and `sudo`. Reading the automount root from `/etc/wsl.conf` or checking an `MZ` header was declined: it parses more to defend a path that is a user-owned config value anyway. |
| D-17 | No `WSL_NOTICE_<KEY>` environment overrides; precedence is flags, config, defaults. | An environment variable that selects the binary or identity is an injection path (a `.envrc`, an injected `export`); nothing here needs it. |
| D-18 | The child gets a built environment (`PATH`, `HOME`, `LANG`, `LC_ALL`, `WSL_DISTRO_NAME`, `WSL_INTEROP`) and working directory `/`. | The caller's environment is not the child's business. |
| D-19 | One wall-clock deadline from the start of input reading; a worker thread plus the main thread's wait, not `SIGALRM`; stop sequence SIGTERM, 2 s, SIGKILL, at most 2 s. See [Toast construction](interface.md#toast-construction). | A signal handler does not run during blocking calls, and per-operation timeouts add up (measured for `http.client` in the sibling notifier's design spikes). |
| D-20 | `validate` never prompts and has no `--yes`; sending a test toast needs `--test-message`; `--probe` runs the version script only. | A prompt blocks automation and doctors; an explicit flag is clearer. |
| D-21 | State problems never fail a send: one stderr warning, no log record. See [Files](interface.md#files). | A toast is not worth failing over a log directory. |
| D-22 | The differing cut marker (`…` here, `…[truncated]` in slack-notice and ha-notice) and the default prefixes (none for `info`) are intentional. | A small surface with a separate title line; slack has one text field and no separate title. |

## Resolved open topics

- Payload shape: [Notification](interface.md#notification).
- Channel config: [Config](interface.md#config); the tool reads no credentials.
- Escaping: handled by construction on the Windows side, plus category-based character removal ([Sanitizer](interface.md#sanitizer)); the script strips a second time before every `InnerText`.

## Live verification

Verified in design (private spike, 2026-11-17, Windows PowerShell 5.1 under WSL2; the tool's own constant script and fixtures):

| Behavior | Result |
|---|---|
| Interop | `powershell.exe` reachable; about 0.8 to 0.95 s per start with `-NoProfile`; no `pwsh.exe`, BurntToast or `wsl-notify-send.exe` present. |
| Toast API | WinRT toast works without a module under the PowerShell identity; exit 0. |
| Text transport | Base64 UTF-8 JSON on stdin; marker text absent from WSL `ps`, `/proc/<pid>/environ` and the Windows command line. |
| Escaping | `InnerText` renders `<`, `&`, `]]>`, quotes and a fake `</text><image …>` literally; NUL and C0 controls make it fail. |
| Limits | 5000-character body, 600-character title and empty body all exit 0. |
| Observability | Display and dismissal not observable from PowerShell. The toast was added to the notification history (history count), but the user saw no toast on screen. |
| Interop error | `UtilAcceptVsock` (exit 1) seen twice, right after a hung or killed call. |
| Control characters | U+FFFE and U+FFFF break `InnerText`, like NUL and C0 controls. |
| Timeout | Terminating the child left no Windows PowerShell process. |

Still to verify (live tests and a human look, conservative defaults until then):

| Id | Claim | Conservative default |
|---|---|---|
| V13 | A toast under the default identity appears on screen (the user saw none in the spike). | The 0.1.0 release gate ([plan.md](plan.md), stage 0); the docs say `sent` only means handed to Windows. |
| V14 | A stale `WSL_INTEROP` is the common cause of `UtilAcceptVsock`; sockets live at `/run/WSL/<pid>_interop`. | Check the inherited value, fall back to the newest live socket, else keep the inherited value; `validate --probe` reports the source. |
| V15 | `UtilAcceptVsock` appears on stderr with exit 1. | Substring on stderr plus exit 1. |
| V16 | A tap on the toast opens `launch`. | Documented as untested. |
| V17 | With interop disabled the exec error is `ENOEXEC`. | `tool_missing` with an interop hint. |
| V19 | A toast `Tag` replaces a toast. | Not used in 0.x. |
| Script | The listing in [Constant script](interface.md#constant-script) runs as written (derived from the spike's script, not run verbatim). | The first live test runs it; the golden test pins it afterwards. |

Also open: how Windows truncates long titles and bodies; Focus Assist behavior; WSL1.

## Modules

| Module | Responsibility |
|---|---|
| `cli.py` | Argument parsing, command dispatch, exit-code mapping, output. |
| `config.py` | Config loading, drop-ins, durations, Pydantic validation (no environment overrides). |
| `payload.py` | Input parsing, notification validation, sanitizer. |
| `render.py` | Title and body assembly, truncation, the JSON line for stdin. |
| `toast.py` | The constant PowerShell script, process start, result mapping, the single retry. |
| `platform.py` | WSL detection, interop socket check and child environment. |
| `log.py` | Log record writing and rotation. |
| `gateway.py` | The only place that starts processes and reads clocks; tests substitute a fake `powershell.exe`. |

## Data model, concurrency, errors

No stored objects and no send lock (callers send one toast at a time); the log takes a short `log.lock`. Errors use the envelope in [Error envelope](interface.md#error-envelope); log failures never change an exit code. The tool signals only the exact child it started and the process group it created.

## Test strategy

- All process starts go through `gateway.py`; a fake `powershell.exe` in `tests/fakebin/` (an absolute `powershell_bin` in the test config) records its argv, stdin, environment and working directory and returns each row of the result table in [Toast construction](interface.md#toast-construction); WSL detection and the interop sockets are faked through the gateway.
- A golden test pins the constant script and asserts the argument list is identical for any notification; tests assert that no notification text appears in argv or the environment, with the spike's marker strings (quotes, `$`, backticks, newlines, emoji, `<`, `&`).
- Golden files for `--json` output and the stdin line, schema tests with positive and negative samples, a sanitizer table test (U+061C, U+2028, U+2029, U+0085, U+E0041, U+00AD, U+FE0F, U+FFFE, U+FFFF, U+FDD0, a U+3164-only message, `a\n \n \nb`, a title with LF and U+2028, a fullwidth space only).
- `live` tests (never in CI, WSL only) show marked toasts and run the injection fixtures through real PowerShell; they need a person to confirm display.

- Further tests: input table, link table and `link_hosts`, duration table, cut table with exact outputs, golden hashes, envelope per code, unusable state directory, deadline with a fake PowerShell that ignores TERM, child environment allowlist and working directory, stale interop socket.

## Platforms

WSL2 on Windows 10 or 11 only; the tool refuses elsewhere.
