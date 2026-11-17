# wsl-notice: design

Status: 0.1.0-design, 2026-11-17. The rules are in [interface.md](interface.md), the single normative source; this document records decisions, modules and tests and links to interface headings without restating them. Architecture: `../awkit-docs/architecture.md` (v1.22).

## Role and layer

L1 primitive: show one Windows toast from a process running inside WSL. Works fully standalone. It is one of three sibling channel notifiers with the same contract (`slack-notice`, `ha-notice`), each designed to be called by a routing tool above it without knowing that tool exists. It was seeded as `desktop-notice`; the maintainer renamed it so the name says what it is, and future platform notifiers (native Windows, other Linux desktops, macOS) are separate tools: `win-notice`, `linux-notice`, `mac-notice`.

## Uses

| Dependency | Layer | Binding | Notes |
|---|---|---|---|
| Windows PowerShell 5.1 through WSL interop | L0 third party | none (not an edge of the tool graph) | One short-lived process per notification; see [Toast construction](interface.md#toast-construction). |
| Python standard library (`subprocess` through `gateway.py`) | L0 | none | No third-party runtime dependency beyond Pydantic and PyYAML. |
| Pydantic (config validation) | library | none | Required by the design guide for config. Pydantic errors are converted to messages built from `loc` and `type` only, because the default message contains the input value. The scaffold's `pyproject.toml` does not list it yet; adding it is part of stage 1 of [plan.md](plan.md). |
| PyYAML (config parsing) | library | none | The design guide mandates YAML config; a strict `yaml.SafeLoader` subclass is used (only `true` and `false` are booleans, only decimal numbers are numbers), on regular files of at most 64 KiB. Added with Pydantic in stage 1. |

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
| D-8 | `link` becomes the toast's protocol-activation target, `http` and `https` only, printable ASCII, one regex whose path and query characters are the RFC 3986 set (callers percent-encode everything else; a valid percent escape is required, one fragment is allowed, `[` and `]` appear only in a bracket host that `ipaddress.IPv6Address` accepts), scheme compared case-insensitively and rendered lowercase; no code calls a URL parser on the link; optional `link_hosts` (entries of at least two labels) restricts hosts and is unset by default. See [Notification](interface.md#notification). | The toast API accepts any scheme (`file:`, `ms-settings:` and custom protocols built without validation). The URL is hidden behind a tap, so printable ASCII removes bidi, zero-width and homograph display problems in one rule (maintainer decision, 2026-11-17). A narrow character class and no second parse leave no parser differential. A numeric or shortened IPv4 host is not refused: with `link_hosts` unset any web address is allowed, and with it set such a host cannot match an entry. |
| D-9 | Output says `sent`, never `displayed`; exit 12 `toast_failed` only when PowerShell fails. | Display, dismissal and Focus Assist suppression are not observable from PowerShell 5.1 (spike C). |
| D-10 | Failures are classified by how far the child got. `interop_failed` (retryable) is only for what proves the toast was not attempted: exit 1 with `UtilAcceptVsock` after the single retry (500 ms, inside the deadline, only when at least 2 s remain), and a deadline before the child was started. Every other end of the child (an unlisted exit, exit 1 without the signature, a kill by signal, a crash after launch, the deadline after launch) is `outcome_unknown`, not retryable, with `details.exit_status`. A stage token needs exit 3. No marker is written before `Show`, so a deadline that passes after launch but before the toast stays `outcome_unknown`. See [Toast construction](interface.md#toast-construction). | A duplicate toast is worse than a lost one (maintainer decision, 2026-11-17), and a permanent host fault (AppLocker, constrained language mode) must not loop as a retryable code. The cost of the single rule is a lost toast on a start slower than the timeout (cold start 1 to 5 s against a default of 10 s; raise `timeout`). A marker protocol could be added later without breaking callers (declined for 0.x by the maintainer, 2026-11-17). The `UtilAcceptVsock` premise is unverified (V15). |
| D-11 | Limits sit below what Windows accepted (title 100, body 1000 by default). | 5000-character bodies were accepted but on-screen truncation is not observable. |
| D-12 | Default `app_id` is the Windows PowerShell identity; unregistered identities are allowed by config but warned about in the docs. | The PowerShell identity needs no registration or install; unregistered ones produced no error and possibly no toast. |
| D-13 | Exit codes and outcomes as in [Exit codes](interface.md#exit-codes); exit 22 for a disabled tool; `not_wsl` is exit 12 (retryable false). | Exit 20 means only that the payload was refused; environment faults are exit 12 codes, so a router has one rule: 20 drop and surface, 12 read the flags, 22 no-op (maintainer decision, 2026-11-17). Same as the sibling notifiers. |
| D-14 | No hooks in 0.x and no audit file; `config_invalid` is exit 2. See [Hooks](interface.md#hooks), [Log](interface.md#log). | As in the sibling notifiers. |
| D-15 | Sanitizer by Unicode category (Cc, Cf, Cs, Co) plus noncharacters and a short filler list, in a fixed order; Cn is not used. See [Sanitizer](interface.md#sanitizer). | Hand-listed ranges missed characters. Cn membership changes with the Unicode version of the running Python, and the ranges listed as fillers are all invisible code points that no legitimate text uses, so they are listed instead of using Cn as a rule. The filler list is checked against the running Python's tables by a test that enumerates `unicodedata` for the default-ignorable ranges (the survivors list is the test's oracle, not a rule). Mn joiners that are legitimate inside text (for example U+2D7F and U+1107F) are not removed; a text made only of whitespace and Mn, Mc and Me characters is empty by definition. The rule also removes the zero-width joiner and variation selector 16, so some emoji sequences render as separate or text-style emoji (accepted). U+FFFE and U+FFFF break `InnerText` (spike). |
| D-16 | `powershell_bin` is an absolute path, default `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe`; no `PATH` search; WSL detected only from `/proc/sys/kernel/osrelease`; `WSL_DISTRO_NAME` ignored. | Cron, systemd and a planted `powershell.exe` earlier in `PATH` become irrelevant; `WSL_DISTRO_NAME` is missing under cron and `sudo`. Reading the automount root from `/etc/wsl.conf` or checking an `MZ` header was declined: it parses more to defend a path that is a user-owned config value anyway. |
| D-17 | No `WSL_NOTICE_<KEY>` environment overrides; precedence is flags, config, defaults. | An environment variable that selects the binary or identity is an injection path (a `.envrc`, an injected `export`); nothing here needs it. |
| D-18 | The child gets a built environment (`PATH`, `HOME`, `LANG`, `LC_ALL`, `WSL_DISTRO_NAME`, `WSL_INTEROP`) and working directory `/`. | The caller's environment is not the child's business. Whether more than `WSL_INTEROP` is needed is open (V22). |
| D-19 | One monotonic deadline from the start of input reading, single-threaded: `Popen`, a selector on the two pipes and `Popen.wait(timeout)`, every wait bounded by the time that remains; no worker thread and no `SIGALRM`. One handler for SIGINT, SIGTERM and SIGHUP records that the deadline has passed. Stop sequence: SIGTERM to the process group, up to 2 s, SIGKILL to the group only while the child is not reaped, up to 2 s more. See [Toast construction](interface.md#toast-construction). | The child phase has no blocking call that a selector and a bounded wait cannot interrupt, so there is no race to close (a worker thread, as an HTTP notifier needs for name resolution, would only add one). The main thread decides the result before it stops the child, so a late exit cannot change it. The group is checked with `poll()` before each signal so a recycled process group id is never signalled. |
| D-20 | `validate` never prompts and has no `--yes`; sending a test toast needs `--test-message`; `--probe` runs the version script only. Plain `validate` ignores `enabled` except to report it; `--probe` and `--test-message` on a disabled tool make no call, print `{"outcome":"disabled"}` and exit 22. | A prompt blocks automation and doctors; an explicit flag is clearer. Exit 22 means the requested action was not performed; plain `validate` is a doctor call and exits by its checks. |
| D-21 | State problems never fail a send: one stderr warning, no log record. Every state file is opened with `O_NOFOLLOW` and `O_NONBLOCK` and must be a regular file; the log lock is polled with `LOCK_NB` for at most 1 s and the record is skipped on failure; the log step is outside the deadline and bounded. See [Files](interface.md#files). | A toast is not worth failing over a log directory, and a FIFO or a held lock must not block the tool. |
| D-22 | The cut marker is `…` and `info` has no default prefix; `max_title` and `max_body` are separate limits; `--dedupe-key` and `--decision-id` are `send` flags. See [Rendering](interface.md#rendering). | A toast has a separate title line, so a small marker and two limits fit; the differences from the other notifiers are in "Why the notifiers differ" below. |
| D-23 | No configuration is selected by the environment: the config path comes from the password database (`pwd.getpwuid(os.getuid()).pw_dir`) and neither `HOME` nor `XDG_CONFIG_HOME` is read; `XDG_STATE_HOME` (an absolute path) still locates the state directory, which otherwise lives under the same home; tests use `--config` and `XDG_STATE_HOME`. | An environment variable that chooses the program that receives every notification is the same injection path as the removed `WSL_NOTICE_<KEY>` overrides (D-17). The cost is that a user with a non-default `XDG_CONFIG_HOME` passes `--config`. This deviates from the design guide's "respect XDG" rule; the deviation is recorded as a carry-forward for the guide (maintainer decision, 2026-11-17). |
| D-24 | Input is read by the main thread in slices of at most 100 ms (gateway `read_source(fd_or_path, max_bytes, remaining)`); a producer that stalls is `invalid_payload` with pointer `""` at the deadline. | The main thread owns the deadline and the signal handler, so a reader thread would add a race for nothing. The 100 ms slice is the cost. |
| D-25 | The interop socket is chosen before every spawn by an owner check (`lstat`: a socket owned by uid 0), a 200 ms connect test, a newest-first fallback among at most five root-owned sockets, and no failure before the spawn. See [Toast construction](interface.md#toast-construction). | `WSL_INTEROP` can name an attacker's listener, and failing without spawning would contradict V14's conservative default. A connect-and-close test is harmless on the one machine checked (V23). |

## Resolved open topics

- Payload shape: [Notification](interface.md#notification).
- Channel config: [Config](interface.md#config); the tool reads no credentials.
- Residual risks: [Residual risks](interface.md#residual-risks) states that the environment is trusted and that `--config` selects the destination.
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
| V14 | A stale `WSL_INTEROP` is the common cause of `UtilAcceptVsock`; sockets live at `/run/WSL/<pid>_interop`. | Keep the inherited value when it is a root-owned socket that accepts; fall back to the newest root-owned socket that accepts; else keep a root-owned inherited value anyway; `validate --probe` reports the source. |
| V15 | `UtilAcceptVsock` appears on stderr with exit 1. | Substring on stderr plus exit 1. |
| V16 | A tap on the toast opens `launch`. | Documented as untested. |
| V17 | With interop disabled the exec error is `ENOEXEC`. | `tool_missing` with an interop hint. |
| V19 | A toast `Tag` replaces a toast. | Not used in 0.x. |
| V22 | The interop child needs only `WSL_INTEROP` (not `PATH`, `HOME`, `LANG`, `LC_ALL`, `WSL_DISTRO_NAME`). | Keep the list; the stage-0 live check runs with `WSL_INTEROP` alone, then the list is reduced. |
| V23 | A connect-and-close test of the interop socket has no side effect. | 200 ms test; 50 connects on one machine left later starts unharmed; the stage-0 live check repeats it. |
| V25 | Interop sockets are owned by root on every supported WSL build. | Require uid 0; an unexpected owner shows as `interop_failed` after the retry. |
| Script | The listing in [Constant script](interface.md#constant-script) runs as written (derived from the spike's script, not run verbatim). | The first stage-0 task runs the listing verbatim once for real (`[Console]::In.ReadLine()` under `-EncodedCommand -NonInteractive` with redirected stdin) and records the result here before stage 3; the golden test pins it afterwards. |

Also open: how Windows truncates long titles and bodies; Focus Assist behavior; WSL1 (not tested).

## Modules

| Module | Responsibility |
|---|---|
| `cli.py` | Argument parsing, command dispatch, exit-code mapping, output. |
| `config.py` | Config loading from the password-database home, drop-ins, strict YAML loader, durations, Pydantic validation (no environment overrides). Stage 1. |
| `payload.py` | Input parsing, notification validation, sanitizer. |
| `render.py` | Title and body assembly, truncation, the JSON line for stdin. |
| `toast.py` | The constant PowerShell script, process start, result mapping, the single retry. |
| `host.py` | WSL detection, interop socket choice and child environment (named `host` because `platform` shadows a standard-library module). |
| `log.py` | Log record writing and rotation, bounded. Stage 1. |
| `gateway.py` | The only place that starts processes, reads clocks, input sources (`read_source`), the home directory and uid (`home_dir()`, `uid()`) and the interop sockets; tests substitute fakes. |

## Data model, concurrency, errors

No stored objects and no send lock (callers send one toast at a time); the log takes a short `log.lock`. Errors use the envelope in [Error envelope](interface.md#error-envelope); log failures never change an exit code. The tool signals only the exact child it started and the process group it created.

## Test strategy

- Process tests call `cli.main(argv, env, gateway)` in-process with a fake gateway (process runner, `/proc/sys/kernel/osrelease` text, `/run/WSL` listing and sockets, clock, home directory and uid, input reading, state-directory hooks); no test-only environment seam is allowed. A small set of real-subprocess tests uses `tests/fakebin/powershell.exe` as an absolute `powershell_bin` (stdin pipe, process-group signals, BrokenPipe).
- All process starts go through `gateway.py`; a fake `powershell.exe` in `tests/fakebin/` records its argv, stdin, environment and working directory and returns each row of the result table in [Toast construction](interface.md#toast-construction); WSL detection and the interop sockets are faked through the gateway.
- A golden test pins the constant script and asserts the argument list is identical for any notification; tests assert that no notification text appears in argv or the environment, with the spike's marker strings (quotes, `$`, backticks, newlines, emoji, `<`, `&`).
- Golden files for `--json` output and the stdin line, schema tests with positive and negative samples, a sanitizer table test (U+061C, U+2028, U+2029, U+0085, U+E0041, U+00AD, U+FE0F, U+FFFE, U+FFFF, U+FDD0, a U+3164-only message, `a\n \n \nb`, a title with LF and U+2028, a fullwidth space only).
- `live` tests (never in CI, WSL only) show marked toasts and run the injection fixtures through real PowerShell; they need a person to confirm display.

- Further tests: input table, link table and `link_hosts`, duration table, cut table with exact outputs, golden hashes, envelope per code, unusable state directory, deadline with a fake PowerShell that ignores TERM, child environment allowlist and working directory, stale interop socket.

## Why the notifiers differ

The notifiers share a contract but not every rule, and each difference has a reason. Timeout floors: 5 s here because a cold start takes 1 to 5 s and the interop retry needs room (the HTTP notifiers have a floor of their own, set by spacing and one retry or by a separate connect phase). `max_title` and `max_body` instead of one `max_chars`: a toast has a separate title line, the chat notifier has one text field. The cut marker is `…` and `info` has no default prefix: a small surface. `link_hosts` exists only where the URL is hidden behind a tap. There is no send lock because nothing throttles a local toast; the caller sends one at a time. There is no tag-conditional retry because the toast API has no replace-by-tag in 0.x. The deadline is single-threaded because the child phase can be bounded by a selector and a timed wait; a notifier that waits on name resolution needs a worker thread.

## Platforms

WSL2 on Windows 10 or 11 only; the tool refuses elsewhere.
