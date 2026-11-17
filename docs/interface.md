# wsl-notice: interface

Status: 0.1.0-design, 2026-11-17. Written before implementation (architecture section 18.1). This document is the single normative source for wsl-notice's rules; `docs/design.md`, the README and the architecture link to its headings and do not restate them.

wsl-notice shows one Windows toast notification from a Linux process running inside WSL, through Windows PowerShell. It is the WSL member of a planned family of platform notifiers: a native Windows tool, other Linux desktops and macOS would be separate tools with their own names (`win-notice`, `linux-notice`, `mac-notice`). It is stateless: it keeps no database, only `log.jsonl`. It runs no hooks and takes no policy decisions: when to send, what to send and how often belong to its caller.

References: architecture (`../awkit-docs/architecture.md`) sections 5.1, 13.2, 15 and 20; design guide (`../awkit-docs/design-guide.md`).

## Concepts

### Notification

A notification is a JSON object ([schemas/notification.json](../schemas/notification.json)):

| Field | Type | Required | Rule |
|---|---|---|---|
| `title` | string | no | One line (newlines become spaces); cut at `max_title` characters after sanitizing ([Rendering](#rendering)). At most 100000 characters of input (else exit 20, reason `invalid_payload`). |
| `message` | string | yes | At most 100000 characters of input (else exit 20, reason `invalid_payload`). Must not be empty after sanitizing (else exit 20, reason `empty_message`). |
| `level` | `info`, `warn`, `action`, `error` | no | Default `info`. Ordered `info < warn < action < error`; the order matters to callers, not to wsl-notice. |
| `link` | string | no | `http` or `https` URL, at most 1024 characters, no whitespace or control characters, no userinfo. Anything else exits 20, reason `link_invalid` (`link_scheme` for another scheme). The schema checks only the syntactic gate (http or https, no userinfo, no whitespace or control characters, at most 1024 characters); the tool names the reason. |

Unknown fields exit 20, reason `invalid_payload` (`details.pointer`). Input is read from stdin as JSON (`send --stdin-json`, also the default when stdin is not a terminal and no text flag is given), from a file (`--file <path|->`), or from flags (`--title`, `--message`, `--level`, `--link`). Flag text is untrusted and goes through the same [sanitizer](#sanitizer) as stdin text; flags exist for humans. Giving both a JSON source and a text flag exits 2 `usage`.

### Sanitizer

Applied to `title` and `message` before rendering, in this order:

1. Invalid UTF-8 or lone surrogates: exit 20, reason `invalid_payload`.
2. CRLF and CR become LF.
3. Control characters (category Cc) except LF are removed (NUL and other C0 characters would make the toast XML builder fail); the tab becomes a space.
4. Invisible and direction-changing format characters are removed: U+200B to U+200F, U+202A to U+202E, U+2060 to U+2064, U+2066 to U+2069, U+FEFF.
5. Runs of more than two LF become two; leading and trailing whitespace is trimmed.

No markup escaping is done in the tool, because the text never becomes markup: the toast is built from text nodes on the Windows side ([Toast construction](#toast-construction)).

### Rendering

- The toast title is the level's entry in `level_prefix` (defaults: none for `info`, `WARN:`, `ACTION:`, `ERROR:`; one-line strings up to 16 characters) joined with a space to the title; with neither, the toast has the message as its only text line.
- The title is limited to `max_title` characters (default 100, 20 to 256) and the message to `max_body` (default 1000, 100 to 5000), in Unicode code points; a longer text is cut and ends with `…`, and the result has `truncated: true`. The limits sit below what Windows is known to accept because on-screen truncation is not observable.
- `link`, when given, is the toast's tap action ([Toast construction](#toast-construction)).

## Commands

### `send`

`wsl-notice send [--stdin-json | --file <path|->] [--title <t>] [--message <m>] [--level <l>] [--link <url>] [--dedupe-key <k>]`

Order, with the exit code of each refusal:

1. Parse flags (exit 2 `usage`).
2. Load config (exit 2 `config_invalid`). `enabled: false` prints `{"outcome": "disabled"}` and exits 22 here, before input is read.
3. Read and validate input (exit 2 `usage` for a conflicting source; exit 20 with reason `invalid_payload`, `empty_message`, `link_invalid` or `link_scheme`).
4. Render ([Rendering](#rendering)). With `--dry-run`, print the result with `rendered` (the JSON handed to PowerShell) and exit 0: nothing is started and nothing is logged.
5. Check the platform: not WSL exits 20, reason `not_wsl`; PowerShell not found exits 12 `tool_missing` (`details.install`). Check the state directory (exit 12 `state_unsafe`).
6. Run PowerShell ([Toast construction](#toast-construction)) and map the result to an exit code ([Exit codes](#exit-codes)).
7. Append the [log record](#log) and print the result.

`--dedupe-key` (grammar `^[A-Za-z0-9._:@/-]{1,128}$`) is accepted for uniformity with other notifiers and recorded (hashed) in the log; it has no effect on the toast in 0.x.

### `validate`

`wsl-notice validate [--probe] [--test-message]`

Offline, it checks the config, that the process runs in WSL and that `powershell_bin` resolves. When the config loads, it prints `{ok, checks: [{name, ok, message}]}` ([schemas/cli/validate.json](../schemas/cli/validate.json)) and exits 0 when every check passes. Otherwise it still prints the document with `ok: false` and exits with the exit code of the first failing check: 20 with reason `not_wsl`, or 12 for a tool or host problem (`tool_missing`, `toast_failed`, `interop_failed`, `outcome_unknown`). When the config cannot load, it prints the [error envelope](#error-envelope) and exits 2 `config_invalid`.

`--probe` also runs a constant script that prints the PowerShell version, to prove interop works and measure start time (reported as the top-level fields `powershell_version` and `start_ms`). `--test-message` (needs `--probe`) then shows one marked test toast (`wsl-notice validate: test message`) after a confirmation on `/dev/tty`, or at once with `--yes`, and reports it as a check named `test_message`. It cannot tell whether the toast was displayed: the user must look.

## Global flags

| Flag | Meaning |
|---|---|
| `--json` | Machine output on stdout ([Output](#output)). |
| `--config <path>` | Config file instead of `~/.config/wsl-notice/config.yaml` (drop-ins are read next to it). |
| `--dry-run` | `send` only: render and print, start nothing, write no log record (`attempts` is null). |
| `--yes` | `validate --test-message` only: skip the confirmation. Never overrides a refusal. |
| `--actor <actor>` | Recorded in the log. `human:<user>` when stdin is a terminal, else `auto:unknown` (design guide section 4.4); a caller passes `auto:<name>`. |
| `--verbose`, `--quiet` | More or less human output on stderr. |
| `--version` | Print `wsl-notice <version>`. |

## Toast construction

wsl-notice starts Windows PowerShell with an argument list that never contains notification text and is the same on every call:

`<powershell_bin> -NoProfile -NonInteractive -EncodedCommand <base64 of the constant script>`

- `powershell_bin` is `powershell.exe` or an absolute path (for example the `System32\WindowsPowerShell\v1.0` copy under `/mnt/c`). A bare name resolves only against absolute `PATH` entries. Windows PowerShell 5.1 is the supported host; PowerShell 7 is not needed and not used.
- The notification travels on stdin as one line: the base64 of a UTF-8 JSON object `{title, body, launch, app_id}` (`title` and `launch` null when absent), followed by a newline. Raw UTF-8 on stdin is not used because the console's input code page corrupts it. The environment is wsl-notice's own, minus nothing it needs; no notification text is placed in it.
- The constant script (kept in the package, covered by a golden test) reads that line, decodes it, sets `$ProgressPreference` to `SilentlyContinue`, builds the toast XML with `XmlDocument` using `InnerText` and `SetAttribute` only (never string concatenation, which allows injected images, looping audio and action buttons), and shows it for `app_id`. Any failure is caught and the script exits 3 after writing a short error name to stderr; success exits 0. The script writes nothing else.
- `link`: when present and `http` or `https`, the toast gets `launch` set to it with `activationType="protocol"`. Windows accepts any scheme there, which is why the tool allowlists.
- `app_id` is the application identity the toast is shown under (config `app_id`, default the Windows PowerShell identity, `{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe`, which needs no registration; the toast shows as coming from "Windows PowerShell"). It matches `^[A-Za-z0-9_.{}\\ -]{1,256}$`. An identity that is not registered gives no error and may show nothing.
- The call ends when PowerShell exits or after `timeout`. On timeout wsl-notice signals the exact child process it started (and its process group, which it created), waits for it, and reports `outcome_unknown`.
- When PowerShell fails with the interop error text `UtilAcceptVsock` (exit 1, seen once after a killed or hung call), the call is retried once immediately.

| PowerShell result | Exit |
|---|---|
| Exit 0 | 0, `sent` (the toast was handed to Windows). |
| Exit 3, or any other exit | 12 `toast_failed` (`details.stage`: the short error name, or `exit_<n>`), `retryable: false`. |
| `UtilAcceptVsock` after the one retry | 12 `interop_failed`, `retryable: true`. |
| Timeout | 12 `outcome_unknown`, `retryable: true`, `unknown_outcome: true`. |
| Binary missing or not executable | 12 `tool_missing`. |

Whether the toast was displayed, dismissed or suppressed (Focus Assist, do-not-disturb) is not observable, so output says `sent`, never `displayed`.

## Output

With `--json`, `send` prints one object ([schemas/cli/send.json](../schemas/cli/send.json)): `{outcome, tool, level, truncated, bytes, duration_ms, attempts, dedupe_key, rendered}`. `outcome` is `sent`, `dry_run` or `disabled`; `bytes` is the size of the JSON handed to PowerShell; `attempts` is 1 or 2 for `sent` and null for a dry run; `rendered` is present only for a dry run. A disabled tool prints `{"outcome": "disabled"}` and nothing else. Without `--json`, human text goes to stderr and stdout is empty.

Errors print the [error envelope](#error-envelope) on stdout with `--json` and a one-line message on stderr otherwise.

## Exit codes

Codes 0–21 follow design guide section 4.2. wsl-notice defines one code of 22 or above.

| Exit | Meaning | Codes (reasons for exit 20) |
|---|---|---|
| 0 | Handed to Windows (`sent`) or a dry run. | |
| 1 | Internal error. | `internal` |
| 2 | Usage or invalid configuration. | `usage`, `config_invalid` |
| 12 | Failed for a reportable reason. | `tool_missing`, `toast_failed`, `interop_failed`, `outcome_unknown`, `state_unsafe` |
| 20 | Refused before sending; code `refused`, `reason` required. | Reasons: `invalid_payload`, `empty_message`, `link_invalid`, `link_scheme`, `not_wsl` |
| 22 | Nothing sent: the tool is disabled in config. | `disabled` (not an error; output `{"outcome": "disabled"}`) |

Exits 10, 11, 13 and 21 are never used. A caller treats exit 22 as a successful no-op.

### Error envelope

`{"error": {"code", "message", "reason", "details", "retryable", "retry_after_s", "unknown_outcome"}}` ([schemas/cli/error.json](../schemas/cli/error.json)); only `code` and `message` are required. `reason` is set on exit 20 (`code: refused`) and null elsewhere. `retryable` and `unknown_outcome` are booleans on exit 12 and null elsewhere; `retry_after_s` is always null here. `details` by code or reason: `config_invalid` → `key`; reason `invalid_payload` → `pointer`; `tool_missing` → `install`; `toast_failed` → `stage`.

| Exit 12 code | `retryable` | `unknown_outcome` |
|---|---|---|
| `tool_missing`, `state_unsafe`, `toast_failed` | `false` | `false` |
| `interop_failed` | `true` | `false` |
| `outcome_unknown` | `true` | `true` |

## Hooks

None in 0.x. wsl-notice shows one toast and takes no policy decisions; gating and routing belong to the caller. A `post-send` hook that notifies on failure could call this tool again and loop, which is the other reason there are none. Adding hooks later is additive.

## Config

`~/.config/wsl-notice/config.yaml` (respecting `XDG_CONFIG_HOME`), then `config.d/*.yaml` in lexical order; maps merge, other lists replace. Schema: [schemas/config.json](../schemas/config.json); example: [config.example.yaml](../config.example.yaml). Invalid config exits 2 `config_invalid` naming the key. Precedence: flags, then environment (scalar keys only), then config, then defaults. There is no per-repo override and the tool reads no credentials.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | `false` makes `send` exit 22. |
| `powershell_bin` | `powershell.exe` | Name or absolute path of Windows PowerShell. |
| `app_id` | the Windows PowerShell identity | Application identity the toast is shown under. |
| `level_prefix` | `{info: "", warn: "WARN:", action: "ACTION:", error: "ERROR:"}` | Title prefixes, at most 16 characters each. |
| `max_title` | `100` | Title limit, 20 to 256. |
| `max_body` | `1000` | Message limit, 100 to 5000. |
| `timeout` | `10s` | Budget for one `send` (PowerShell takes about one second), 2s to 60s. A number of seconds or a string like `10s`, `1m` or `500ms`; a value out of range exits 2 `config_invalid` (`details.key`). |
| `log` | `true` | `false` writes no `log.jsonl`. |

## Environment variables

`WSL_NOTICE_<KEY>` sets a scalar config key (`WSL_NOTICE_ENABLED`, `WSL_NOTICE_POWERSHELL_BIN`, `WSL_NOTICE_APP_ID`, `WSL_NOTICE_MAX_TITLE`, `WSL_NOTICE_MAX_BODY`, `WSL_NOTICE_TIMEOUT`, `WSL_NOTICE_LOG`); `level_prefix` cannot be set from the environment. The tool reads `WSL_DISTRO_NAME` and `/proc/sys/kernel/osrelease` to detect WSL. It reads no credentials.

## Log

`~/.local/state/wsl-notice/log.jsonl` (respecting `XDG_STATE_HOME`), mode 0600, appended without following symlinks; one record per `send` that reaches step 3 or later and ends in `sent`, `refused` (exit 20), `failed` (exit 12 except an unknown outcome) or `outcome_unknown` ([schemas/log.json](../schemas/log.json)). Exit 1, exit 2, exit 22 (`disabled`) and dry runs write no record. Fields: `{ts, tool, version, outcome, exit, code, reason, level, bytes, truncated, duration_ms, attempts, message_sha256, dedupe_key_sha256, actor, actor_source}`; every field is always present and nullable ones are null when unknown. `outcome` is `sent`, `refused`, `failed` or `outcome_unknown`; `code` is `refused` for exit 20, the error code for exit 12 and null on success; `reason` is null unless the outcome is `refused`; `actor_source` is `flag` (from `--actor`), `tty` (stdin is a terminal) or `default` (`auto:unknown`). `message_sha256` and `dedupe_key_sha256` are the first 16 hex characters of the SHA-256 of the notification JSON and of the key. Records never contain a title, message or link. The file rotates at 1 MiB, keeping 3 files. A log write failure warns on stderr and never changes the exit code. There is no audit file: a toast changes nothing in a system of record.

## Files

| Path | Content |
|---|---|
| `~/.config/wsl-notice/config.yaml`, `config.d/*.yaml` | Config. |
| `~/.local/state/wsl-notice/log.jsonl` (and `.1` to `.3`) | Log records. |

`~/.local/state/wsl-notice/` is created with mode 0700, opened without following symlinks and checked to be a directory owned by the user with mode 0700 on every send (else exit 12 `state_unsafe`). There is no database and no migration, and no file is written to the Windows side.

## Labels

wsl-notice creates no long-lived objects and takes no labels.

## TUI

None in v1.

## Residual risks

- **No display confirmation.** `sent` means PowerShell returned success. An unregistered `app_id`, Focus Assist, do-not-disturb or notification settings can suppress the toast with no error; whether the default identity shows on a given Windows setup is verified only by a person looking ([design.md](design.md), Live verification).
- **Start cost and interop flakiness.** Each call starts a Windows process (about one second) and can fail once with an interop error after a hung or killed call; one automatic retry covers the case seen so far.
- **The script is trusted, the text is not.** The only defense for the text is that it reaches PowerShell on stdin and becomes XML text nodes; a change to the constant script that concatenated strings into XML would reopen image, audio and action injection (a golden test pins the script).
- **Links open in the default handler.** `link` is limited to `http` and `https`; the toast API itself accepts any protocol.
- **Same-user visibility.** Another Windows or WSL process of the same user can read the notification from the Windows notification history; callers must not send secrets.
- **Not supported:** native Windows, Linux desktop and macOS notifiers (separate tools), PowerShell 7, WSL1 (unverified), custom icons, sounds, buttons, images and progress bars.

## Platform notes

WSL2 on Windows 10 or 11 with Windows PowerShell 5.1 reachable through interop is the only supported platform. The tool refuses elsewhere with exit 20, reason `not_wsl`.
