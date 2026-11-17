# wsl-notice: interface

Status: 0.1.0-design, 2026-11-17. Written before implementation (architecture section 18.1). This document is the single normative source for wsl-notice's rules; `docs/design.md`, the README and the architecture link to its headings and do not restate them. The first release (0.1.0) is gated on a person seeing a toast under the default identity ([plan.md](plan.md), stage 0).

wsl-notice shows one Windows toast notification from a Linux process running inside WSL, through Windows PowerShell. Other platforms (a native Windows tool, other Linux desktops, macOS) would be separate tools with their own names (`win-notice`, `linux-notice`, `mac-notice`). It is stateless: it keeps no database, only `log.jsonl`. It runs no hooks and takes no policy decisions: when to send, what to send and how often belong to its caller.

References: architecture (`../awkit-docs/architecture.md`) sections 5.1, 13.2, 15 and 20; design guide (`../awkit-docs/design-guide.md`).

## Concepts

### Notification

A notification is a JSON object ([schemas/notification.json](../schemas/notification.json)):

| Field | Type | Required | Rule |
|---|---|---|---|
| `title` | string | no | One line (the sanitizer turns newlines into spaces); cut at `max_title` characters after sanitizing ([Rendering](#rendering)). At most 100000 characters of input (else exit 20, reason `invalid_payload`). |
| `message` | string | yes | At most 100000 characters of input (else exit 20, reason `invalid_payload`). Must not be empty after sanitizing or absent (else exit 20, reason `empty_message`). |
| `level` | `info`, `warn`, `action`, `error` | no | Default `info`. Ordered `info < warn < action < error`; the order matters to callers, not to wsl-notice. |
| `link` | string | no | 1 to 1024 printable ASCII characters, `http` or `https`; see the link rule below. |

**Input.** Exactly one source is used: JSON on stdin (`send --stdin-json`, also the default when stdin is not a terminal and no text flag is given), JSON from `--file <path>` (`--file -` is stdin, the same source), or flags. These are flag and combination errors and exit 2 `usage` at step 1, before the config is consulted, so they win over a disabled tool: a JSON source together with `--title`, `--message`, `--level` or `--link`; `--stdin-json` together with `--file <path>`; `--stdin-json` or `--file -` when stdin is a terminal; stdin that is a terminal with no source and no `--message`; flags without `--message`; an unknown flag, a missing value, a bad `--actor`. A `--file` that does not exist, cannot be read or is not a regular file (the tool opens it with `O_NONBLOCK` and checks the opened descriptor; a symlink to a regular file is fine) exits 2 `usage` at step 3. A closed or unreadable stdin is empty input.

Input is read through the gateway in slices of at most 100 ms, so the main thread notices the deadline and a signal between slices; a regular `--file` is read in chunks of 64 KiB with the same checks between chunks. At most 1048576 bytes are read from a source. More, input that is not valid UTF-8 or that begins with a byte order mark, JSON that fails to parse (any `ValueError`, including the digit limit for very long integers, and `RecursionError` from nesting), repeats an object key or contains NaN or Infinity, a top-level value that is not an object, or input that does not arrive within the deadline exits 20 refused, reason `invalid_payload`, `details.pointer` `""`. The last case includes a producer that stalls: the payload never arrived, the caller owns the resend.

Fields: an unknown field, a field of the wrong type (null is a wrong type: absent fields are omitted), a level outside `info`, `warn`, `action`, `error`, or a title or message of more than 100000 characters exits 20 `invalid_payload` with `details.pointer` set to `/<field>` (a field name that does not match `[A-Za-z0-9_-]{1,64}` is reported as `/?`). A flag value is checked like the same JSON field and uses the same pointer. A message that is absent or empty after sanitizing is `empty_message` (reason `empty_message`, no pointer; the schema has no `minLength` for it). Error messages and details never contain an input value, only the pointer and the rule.

The tool may exit without reading all of stdin (a refusal before step 3, an over-size input, any exit 20), so a caller ignores EPIPE and SIGPIPE on the stdin write.

Flag text is untrusted and goes through the same [sanitizer](#sanitizer) as stdin text; flags exist for humans. The deadline is described under [Toast construction](#toast-construction).

**Link rule.** `link` is optional and is checked in this order. (1) It is a string of 1 to 1024 printable ASCII characters (U+0021 to U+007E); anything else, including whitespace, control characters, non-ASCII characters and an empty string, is reason `link_invalid` (callers percent-encode paths and give hosts as punycode). (2) If it starts with a scheme token (`[A-Za-z][A-Za-z0-9+.-]*:`) whose lowercase form is not `http` or `https`, the reason is `link_scheme`. (3) It must match, as a whole, `(?i:https?)://HOST(?::[0-9]{1,5})?(?:[/?]T*)?(?:#T*)?` where `HOST` is `(?:[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*|\[[0-9A-Fa-f:.]+\])` and `T` is `(?:[A-Za-z0-9\-._~:/?@!$&'()*+,;=]|%[0-9A-Fa-f]{2})`; a port must be 1 to 65535 (taken from the match, never from a URL parser), and the text between the brackets must be a valid IPv6 address (`ipaddress.IPv6Address`); else the reason is `link_invalid`. There is no userinfo, empty host, backslash, quote, angle bracket, brace, bar, caret, backtick or invalid percent escape. The link is not parsed again: no code calls a URL parser on it. The scheme is lowercased wherever the link is rendered. The schema pattern is a coarse gate (printable ASCII, 1 to 1024); the tool is authoritative. A link names a place to open and is never a promise that it is safe: a caller that builds links from untrusted text takes that risk.

When config `link_hosts` is set, the link's host (lowercased, without port; a bracket host is never matched, so it is refused) must equal an entry or end with `.` plus an entry, else exit 20 `link_invalid` with `details.pointer` `/link`. An entry matches itself and every subdomain, so do not list a shared hosting domain. With `link_hosts` unset a producer can aim the tap target at any web address; the toast does not show the URL. `link_hosts` is the only protection against an arbitrary tap target.

### Sanitizer

Applied to title and message before rendering, in this order. Every pattern in this document is matched against the whole string (`re.fullmatch`); a trailing newline never passes.

1. Input is read as bytes and decoded as strict UTF-8, never through the locale. Invalid UTF-8 or a lone surrogate: exit 20 refused, reason `invalid_payload`.
2. CRLF, CR, U+0085, U+2028 and U+2029 become LF. The tab becomes a space.
3. These characters are removed: every character of category Cc other than LF, and of categories Cf, Cs and Co; the noncharacters U+FDD0 to U+FDEF and U+FFFE and U+FFFF of every plane; and these invisible fillers: U+034F, U+115F, U+1160, U+17B4, U+17B5, U+180B to U+180F, U+2065, U+2800, U+3164, U+FE00 to U+FE0F, U+FFA0, U+FFF0 to U+FFF8, U+16FE4 and U+E0000 to U+E0FFF. This removes zero-width, direction and tag characters, the soft hyphen and the byte order mark. It also removes the zero-width joiner and variation selector 16, so some emoji sequences are shown as separate or text-style emoji.
4. Trailing whitespace of every line is removed (`str.rstrip`), then runs of more than two LF become two, then leading and trailing whitespace is removed (`str.strip`, Unicode whitespace).
5. title only: every LF becomes a space, runs of spaces become one space, and the result is stripped again.

A text is empty when it has no character other than Unicode whitespace and characters of the categories Mn, Mc and Me. A message that is empty after step 4 is refused with reason `empty_message` (so is a message that is absent from the JSON object). A title that is empty after step 5, or absent, is absent.

No markup escaping is done in the tool, because the text becomes XML text nodes on the Windows side and never markup ([Toast construction](#toast-construction)). As a second line of defense the constant script removes `[\x00-\x08\x0B\x0C\x0E-\x1F\uFFFE\uFFFF]` from `title` and `body` before every `InnerText` assignment (U+FFFE and U+FFFF break `InnerText`; `launch` and `app_id` are ASCII by the rules above and are not stripped).

### Rendering

- The toast title is the level's entry in `level_prefix` (defaults: none for `info`, `WARN:`, `ACTION:`, `ERROR:`; one-line strings up to 16 characters) joined with a space to the title; with neither, the toast has the message as its only text line.
- The title is limited to `max_title` characters (default 100, 20 to 256) and the message to `max_body` (default 1000, 100 to 5000), in Unicode code points. The limits sit below what Windows is known to accept because on-screen truncation is not observable. The rendered title is at most 273 characters.
- `link`, when given, is the toast's tap action ([Toast construction](#toast-construction)); its scheme is lowercased.

Limits count Unicode code points. A title longer than its limit L is cut to its first L-1 characters plus `…` (L in all, before the level prefix is added) and `truncated` is true. The prefix is added after the cut and does not count. A message that does not fit its budget is cut: the kept text is the longest prefix of the sanitized message such that kept, with trailing whitespace removed, plus the marker fits the limit; `truncated` is true. A cut may split a combining or emoji sequence; that is accepted. When the title is empty and the prefix is not, the title line is the prefix alone with no trailing space; when both are empty the title is absent.

The marker is `…` for both `max_title` and `max_body`. `message_sha256` and `bytes` are defined under [Log](#log) and [Output](#output).

## Commands

### `send`

`wsl-notice send [--stdin-json | --file <path|->] [--title <t>] [--message <m>] [--level <l>] [--link <url>] [--dedupe-key <k>] [--decision-id <id>]`

Order, with the exit code of each refusal:

1. Parse flags and check flag combinations (exit 2 `usage`).
2. Load config (exit 2 `config_invalid`). `enabled: false` prints `{"outcome":"disabled"}` (with `--json`) and exits 22 here, before input is read.
3. Read and validate input; the deadline starts here ([Toast construction](#toast-construction)). Exit 2 `usage` for a `--file` problem; exit 20 with reason `invalid_payload`, `empty_message`, `link_invalid` or `link_scheme`.
4. Render ([Rendering](#rendering)). With `--dry-run`, print the result with `rendered` (the JSON handed to PowerShell) and exit 0: steps 5 and 6 are skipped (so a dry run succeeds on any platform), nothing is started and nothing is logged.
5. Check the platform: not WSL exits 12 `not_wsl`; then the binary: `powershell_bin` missing or not executable exits 12 `tool_missing` (`details.install`).
6. Run PowerShell ([Toast construction](#toast-construction)) and map the result to an exit code ([Exit codes](#exit-codes)).
7. Append the [log record](#log) and print the result.

Every JSON document that the tool sends, hashes or measures is serialized with `json.dumps(obj, ensure_ascii=False, separators=(",", ":"))`, keys in the order given in this document, and encoded as UTF-8. `bytes` in the output and the log is the length of the JSON handed to PowerShell, before base64. The key order of that JSON is `title`, `body`, `launch`, `app_id`.

`message_sha256` is the first 16 hex characters of the SHA-256 of the UTF-8 bytes of the sanitized title (empty string when absent), a LF, and the sanitized message, before any cut, prefix or link. `dedupe_key_sha256` is the first 16 hex characters of the SHA-256 of the key's UTF-8 bytes. Both are null when not computed (a refusal before the text was sanitized, or no key).

`--dedupe-key` takes any non-empty string of at most 1024 characters (valid UTF-8, else exit 2 `usage`); `--dedupe-key=<k>` is the same flag, and a key that starts with `-` must use the `=` form. It is only hashed; the send output echoes the caller's own key or null. The dedupe key changes nothing but the log in 0.x: a retry after `outcome_unknown` can show a second toast. `--decision-id` (repeatable) is accepted for uniformity with the design guide and ignored: the log is hash-only and there is no audit file.

`--actor` must match `^(human|auto):[A-Za-z0-9._@/-]{1,64}$` (else exit 2 `usage`). Without it the actor is `human:<user>` when stdin is a terminal (`actor_source` `tty`) and `auto:unknown` otherwise (`default`), where `<user>` is the login name from the password database with every character outside `[A-Za-z0-9._@-]` replaced by `_` and cut to 64 characters; the design guide's `me.name` does not exist here because the tool has no `me` key.

### `validate`

`wsl-notice validate [--probe] [--test-message]`

`validate` never prompts. It prints `{ok, enabled, checks: [{name, ok, message}]}` ([schemas/cli/validate.json](../schemas/cli/validate.json)), where `enabled` is the configured value; plain `validate` ignores `enabled` except to report it, so a tool with `enabled: false` still reports its config and environment faults. Checks run in the order listed below; a failed check ends the run and later checks are omitted, so `ok` is true only when every printed check is ok. It exits 0 when `ok` is true, otherwise with the exit code of the failing check (`config`: 2; every other check: 12). When the config cannot be loaded it prints the error envelope instead and exits 2. Without `--json` it prints one line per check on stderr.

`--test-message` sends one notification through the normal render and send path (level `info`, title `wsl-notice`, message `validate: test message`; no dedupe key) and adds the check `test_message`; its message says that delivery is not observable and tells the user to look at the screen. It is a send: it is logged like a send. A failed `test_message` check carries only the exit code (12), so a caller treats any non-zero exit as "look at the destination". `--probe` makes only the calls that send no notification. With `enabled: false`, `--probe` and `--test-message` make no call: `validate` prints `{"outcome":"disabled"}` ([schemas/cli/send.json](../schemas/cli/send.json)) and exits 22 before any check, as `send` does at step 2. For `--probe` and `--test-message` the deadline and `duration_ms` start when that check starts; `--dry-run` has no effect on `validate`. Automation does not call `--test-message`; a doctor runs `validate --json [--config <path>]` and reads `enabled`.

The checks, in order, are `config`, `platform` (`not_wsl`), `powershell` (`tool_missing`; its message names the resolved `powershell_bin`), `probe` and `test_message`. `--probe` runs the constant probe script ([Constant script](#constant-script)) once under the same deadline, stop sequence and `UtilAcceptVsock` retry as a send: it proves interop works and measures start time, and adds the top-level fields `powershell_version` and `start_ms`, which are present only when the probe check is ok. `start_ms` is `round((t_end_of_stdout - t_spawn) * 1000)` on the gateway clock. The probe check message says which of the four interop sources was used (`inherited`, `fallback`, `inherited unchecked`, `none`; see [Toast construction](#toast-construction)). Any probe failure (spawn error, non-zero exit, output that does not match, interop failure, deadline) fails the `probe` check with exit 12 and a fixed message per cause (tool missing, interop, unexpected probe output, timeout).

## Global flags

Global flags may appear before or after the subcommand. Flags are exact: abbreviations of long options are not accepted.

| Flag | Meaning |
|---|---|
| `--json` | Machine output on stdout ([Output](#output)). |
| `--config <path>` | Config file instead of `.config/wsl-notice/config.yaml` under the home directory; drop-ins are read from `config.d/` beside it. |
| `--dry-run` | `send` only: render and print, start nothing, write no log record (`attempts` is null). |
| `--actor <actor>` | Recorded in the log; grammar in the hashing rule under [`send`](#send). |
| `--verbose`, `--quiet` | See [Output](#output). |
| `--version` | Print `wsl-notice <version>`. |

`--dedupe-key` and `--decision-id` are `send` flags ([`send`](#send)).

## Toast construction

wsl-notice starts Windows PowerShell with an argument list that never contains notification text and is the same on every call:

`<powershell_bin> -NoProfile -NonInteractive -EncodedCommand <base64 of the constant script>`

- `powershell_bin` is an absolute path (config; default `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe`). There is no `PATH` search and no environment override: a user whose automount root differs sets it in config. Windows PowerShell 5.1 is the supported host; PowerShell 7 is not needed and not used.
- WSL is detected only by `/proc/sys/kernel/osrelease` containing `microsoft` (case-insensitive), read through the gateway; `WSL_DISTRO_NAME` is ignored. WSL1 passes the check and is not distinguished (not tested).
- The notification travels on stdin as one line: the base64 of a UTF-8 JSON object `{title, body, launch, app_id}` (`title` and `launch` null when absent), followed by a newline. Raw UTF-8 on stdin is not used because the console's input code page corrupts it. `app_id` is only ever passed inside that JSON, never in a command line.
- The child's environment is built, not inherited: only `PATH`, `HOME`, `LANG`, `LC_ALL`, `WSL_DISTRO_NAME` and `WSL_INTEROP` are copied, and only when set; `WSLENV` is not passed. The working directory is `/`. No notification text is placed in the environment or the argument list. Whether the child needs more than `WSL_INTEROP` is unverified (design.md, V22); the list is kept until the stage-0 live check shows the call works with `WSL_INTEROP` alone, then it is reduced.
- Before every spawn, including the retry, the tool chooses the `WSL_INTEROP` value for the child. A candidate is acceptable when `lstat` says it is a socket (not a symlink) owned by uid 0 and a connection to it succeeds within 200 ms (the connection is closed at once; the check counts against the deadline). Order: (1) the inherited `WSL_INTEROP` when it is acceptable; (2) otherwise the acceptable socket with the newest `st_mtime` among at most five regular sockets named `/run/WSL/<digits>_interop` (symlinks skipped; equal times are ordered by name); (3) otherwise, when the inherited value names a socket owned by uid 0 that did not accept (a stale value or a misjudged test), the inherited value anyway, so the single retry decides; (4) otherwise (unset, or not a root-owned socket) the variable is not passed. There is no failure before the spawn. `validate --probe` says which of the four was used (`inherited`, `fallback`, `inherited unchecked`, `none`). This is best effort (design.md, V14).
- `link`: when present and valid, the toast gets `launch` set to it with `activationType="protocol"`; the tool does no escaping, because the XML API escapes attribute text. Windows accepts any scheme there, which is why the tool allowlists.
- `app_id` is the application identity the toast is shown under (config `app_id`, default the Windows PowerShell identity, `{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe`, which needs no registration; the toast should show as coming from "Windows PowerShell"). It matches `^[A-Za-z0-9_.{}\\ -]{1,256}$`. An identity that is not registered gives no error and may show nothing.
- On failure the script writes one fixed token to stderr and exits 3: `decode`, `json`, `xml` or `show`, naming the step that was running, and writes nothing else. `details.stage` is that token when stderr, trimmed, is exactly one of them and the exit status is 3, and `unknown` when the exit was 3 with any other stderr. A token with another exit status counts as "any other exit". Other stderr text is discarded (never printed or logged). Stderr is read up to 4 KiB.

**Deadline.** Config `timeout` is one monotonic deadline for the whole send, counted from the start of input reading (step 3) to the end of step 6. It covers reading input, the interop socket check, starting the child and waiting for it. There is no worker thread: the main thread reads the child's stdout and stderr with a selector, writes the stdin line without blocking, and waits with the time that remains. At the deadline, or at the first SIGINT, SIGTERM or SIGHUP (later signals are ignored while the result is reported), the main thread decides the result, then stops the child, then writes the log record, prints the output and exits.

The result is `outcome_unknown` (`details.attempts`, `details.exit_status` null) when the child was started and has not ended with a result; `interop_failed` (`details.attempts` null) when the deadline passed before the child was started; `invalid_payload` during input. Stopping the child: if the child has not been reaped, SIGTERM to its process group (which the tool created); wait up to 2 s, ending early when the child exits; if it has still not been reaped, SIGKILL to the group; wait at most 2 s more; the result stands whether or not the child has gone (the message then says the child may still run). The group is signalled only while the child has not been reaped (checked with `poll()` before each signal), so a recycled process group id is never signalled. A BrokenPipeError (also on closing stdin or flushing) is never an error by itself: the exit status of the child decides. The tool ends no later than the deadline plus 4 s plus the bounded log step ([Files](#files)). `timeout` is 5s to 60s (default 10s): a cold start takes 1 to 5 s and the interop retry needs room.

**Interop retry.** An exit 1 whose captured stderr (cap 4 KiB, case-sensitive substring) contains `UtilAcceptVsock` proves the toast script did not start. It is retried once after 500 ms, inside the same deadline and only when at least 2 s remain; if no retry is possible, or the second attempt ends the same way, the result is `interop_failed` (`details.attempts` 1 or 2). The second attempt is classified by the table below. `attempts` is the number of PowerShell spawns made (null when none was made: a dry run, `not_wsl`, `tool_missing`, a spawn error, a deadline before the spawn).

The tool reads the child's stdout and stderr with a selector while it writes the stdin line (the line is at most about 28 KiB, below the pipe size, and is written without blocking); stderr is kept up to 4 KiB, stdout up to 4 KiB (drained and ignored for `send`; used by the probe).

| PowerShell result | Outcome |
|---|---|
| Exit 0 | Exit 0, `sent` (the toast was handed to Windows). |
| Exit 3 | Exit 12 `toast_failed`, `details.stage`, `details.attempts`. |
| Exit 1 with `UtilAcceptVsock` | The retry above; then exit 12 `interop_failed`. |
| Any other exit status (including exit 1 without the signature, an exit with a stage token but a status other than 3, and a kill by signal, shown as a negative status), or the child ending with no result | Exit 12 `outcome_unknown`, `details.attempts`, `details.exit_status` (the status, negative for a signal). |
| Spawn error `ENOENT`, `EACCES`, `ENOTDIR` or `ENOEXEC` | Exit 12 `tool_missing` (`details.install`). Any other spawn error is exit 1. |
| The deadline passed before the child was started | Exit 12 `interop_failed`, `details.attempts` null. |
| The deadline passed after the child was started | Exit 12 `outcome_unknown`, `details.attempts`, `details.exit_status` null. |

`retryable` and `unknown_outcome` per code are in the [Error envelope](#error-envelope). A duplicate toast is worse than a lost one, so every end of the child that is not a known failure is `outcome_unknown`, which is not retryable: this includes a deadline that passes after PowerShell started but before the toast was shown. Whether the toast was displayed, dismissed or suppressed (Focus Assist, do-not-disturb) is not observable, so output says `sent`, never `displayed`.

### Constant script

The script text is constant, kept in the package, encoded into `-EncodedCommand` (base64 of the UTF-16LE text, stored with LF line ends) and pinned by a golden test (the listing, the encoded argument and the XML skeleton). The listing is ASCII only, and a test asserts it contains no non-ASCII byte. It is specified here:

```powershell
$ErrorActionPreference = 'Stop'
$ProgressPreference = 'SilentlyContinue'
$stage = 'decode'
try {
  $raw = [Console]::In.ReadLine()
  if ([string]::IsNullOrEmpty($raw)) { throw 'empty' }
  $json = [System.Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($raw))
  $stage = 'json'
  $d = $json | ConvertFrom-Json
  $strip = '[\x00-\x08\x0B\x0C\x0E-\x1F\uFFFE\uFFFF]'
  $body = [regex]::Replace([string]$d.body, $strip, '')
  $title = $null
  if ($null -ne $d.title) { $title = [regex]::Replace([string]$d.title, $strip, '') }
  $stage = 'xml'
  [void][Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom.XmlDocument, ContentType = WindowsRuntime]
  [void][Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime]
  $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
  $xml.LoadXml('<toast><visual><binding template="ToastGeneric"><text/><text/></binding></visual></toast>')
  $t = $xml.GetElementsByTagName('text')
  $second = $t.Item(1)
  if ($null -ne $title) {
    $t.Item(0).InnerText = $title
    $second.InnerText = $body
  } else {
    $t.Item(0).InnerText = $body
    [void]$second.ParentNode.RemoveChild($second)
  }
  if ($null -ne $d.launch) {
    $r = $xml.DocumentElement
    $r.SetAttribute('launch', [string]$d.launch)
    $r.SetAttribute('activationType', 'protocol')
  }
  $stage = 'show'
  $toast = New-Object Windows.UI.Notifications.ToastNotification($xml)
  [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier([string]$d.app_id).Show($toast)
} catch {
  [Console]::Error.Write($stage)
  exit 3
}
exit 0
```

The `\uFFFE\uFFFF` in the `$strip` line are backslash-u escapes that the .NET regex engine interprets; the file never contains the noncharacters themselves.

Rules the listing must keep, whatever its formatting: one line is read from stdin and an empty line is stage `decode`; text reaches the XML only through `InnerText`, and `launch` only through `SetAttribute` on the root `toast` element; there are no `audio`, `actions`, `image` or `duration` elements; every failure writes its stage token to stderr and exits 3, and nothing else is written. The stage token names the step that was running (a valid non-object JSON value fails under `show`). `ConvertFrom-Json` turns a string `\/Date(1700000000000)\/` into a date; `json.dumps` never emits `\/`, so the tool cannot trigger it, and one golden case pins that.

The probe script used by `validate --probe` is also constant: it prints `$PSVersionTable.PSVersion.ToString()` and exits 0, and the tool accepts only output that matches `[0-9]+(\.[0-9]+){1,3}` as a whole after one trailing newline. A second golden test shows the XML produced for injection payloads (images, audio, actions, a `launch` with quotes) and asserts only text nodes and one attribute.

## Output

With `--json`, `send` prints one object ([schemas/cli/send.json](../schemas/cli/send.json)): `{outcome, tool, level, truncated, bytes, duration_ms, attempts, dedupe_key, rendered}`. `outcome` is `sent`, `dry_run` or `disabled`; `bytes` is the size of the JSON handed to PowerShell before base64; `attempts` is 1 or 2 for `sent` and null for a dry run; `rendered` is present only for a dry run. A disabled tool prints `{"outcome":"disabled"}` and nothing else.

Global flags may appear before or after the subcommand. Flags are exact: abbreviations of long options are not accepted, and `--flag=value` is the same flag as `--flag value` (a value that starts with `-` must use the `=` form). The tool looks for `--json` in argv before it parses anything, so a usage error with `--json` prints the envelope (code `usage`) on stdout; `--help` and `--version` are exempt. Errors always print one line on stderr; with `--json` they also print the error envelope on stdout. With `--json` a successful send prints one object on stdout; without it human text goes to stderr and stdout is empty (a dry run prints the rendered text on stderr). A disabled tool prints `{"outcome":"disabled"}` on stdout with `--json` and one line on stderr without it, and exits 22 either way. stderr is advisory (warnings, verbose lines); the contract is the exit code plus the stdout JSON.

stdout and stderr are written as UTF-8 bytes with errors replaced, never through the locale. After step 6 a failure to write (BrokenPipeError or any OSError) is ignored and the process ends with the exit code of the result after one flush attempt of each stream.

Without `--json` the human lines on stderr are fixed: `sent (<level>, <duration_ms> ms)` for a send, the rendered text for a dry run, `disabled in config`, and `<code>: <message>` for an error (`refused (<reason>): <message>` for exit 20). They contain no title, message, link, dedupe key or other input-derived text. `--quiet` suppresses every stderr line except errors and warnings; `--verbose` adds one line per phase (input, render, send, log) with the elapsed milliseconds and never a request line, header, URL, host, body or credential.

## Exit codes

Codes 0 to 21 follow design guide section 4.2. Exit 0: sent or a dry run. Exit 1: internal error. Exit 2: usage or invalid configuration (codes `usage`, `config_invalid`). Exit 12: failed for a reportable reason (the codes of the tool, below). Exit 20: refused before sending because of the payload; code `refused` with a required reason. Exit 22: nothing sent because the tool is disabled in config (not an error; output `{"outcome":"disabled"}`). Exits 10, 11, 13 and 21 are never used: the tool stores no objects, has no hooks and never prompts. A caller treats exit 22 as a successful no-op.

| Exit | Meaning | Codes (reasons for exit 20) |
|---|---|---|
| 0 | Handed to Windows (`sent`) or a dry run. | |
| 1 | Internal error. | `internal` |
| 2 | Usage or invalid configuration. | `usage`, `config_invalid` |
| 12 | Failed for a reportable reason. | `not_wsl`, `tool_missing`, `toast_failed`, `interop_failed`, `outcome_unknown` |
| 20 | Refused before sending because of the payload; code `refused`, `reason` required. | Reasons: `invalid_payload`, `empty_message`, `link_invalid`, `link_scheme` |
| 22 | Nothing sent: the tool is disabled in config. | `disabled` (not an error; output `{"outcome":"disabled"}`) |

Exit 20 is only for payload refusals, never retried; environment faults (not WSL, no PowerShell) are exit 12 codes, so a caller has one rule: 20 drop and surface, 12 read the flags, 22 no-op.

### Error envelope

The error envelope is `{"error": {"code", "message", "reason", "details", "retryable", "retry_after_s", "unknown_outcome"}}` ([schemas/cli/error.json](../schemas/cli/error.json)); only `code` and `message` are required and the other keys are present with null when they do not apply. `message` is a fixed string of the tool, never a response body, header, exception text or input value. `reason` is set only on exit 20. `retry_after_s` is always null here (the key stays for a uniform shape). `retryable` means that sending the same notification again is safe. `unknown_outcome` means that the notification may already have been delivered. They are set as follows.

- Exit 2 and exit 20: `retryable` null, `unknown_outcome` null.
- Exit 1: `retryable` false; `unknown_outcome` true when the failure happened after the child was started, else false. The envelope on exit 1 is best effort: exit 1 with no envelope means the outcome is unknown.
- Exit 12: both are booleans, set per code in the table below.

`details` by code or reason, null when none applies: `config_invalid`: `key` (a config key name matching `[A-Za-z0-9_]{1,64}`, else `?`; `--config` for a missing `--config` file; or the file path for a file that cannot be read or parsed); reason `invalid_payload`: `pointer`; reason `link_invalid` from `link_hosts`: `pointer` `/link`; `toast_failed`: `stage`, `attempts`; `interop_failed`: `attempts` (null when no child was started); `outcome_unknown`: `attempts`, `exit_status`; `tool_missing`: `install`. `details.install` is the fixed text "Set powershell_bin to the absolute path of Windows PowerShell 5.1, enable interop ([interop] enabled=true in /etc/wsl.conf) and check the automount root." A message never contains an input value, and no stderr line contains input-derived text.

Exit 12 booleans: not sent and permanent (`retryable` false, `unknown_outcome` false): `not_wsl`, `tool_missing`, `toast_failed`; not sent and worth retrying (true, false): `interop_failed`; possibly shown (false, true): `outcome_unknown`. The `interop_failed` message is "PowerShell could not use the WSL interop socket". `details.attempts` is the number of PowerShell spawns made (1 or 2); `details.exit_status` is the exit status of the child (negative for a signal) or null when the deadline ended it.

| Exit 12 code | `retryable` | `unknown_outcome` |
|---|---|---|
| `not_wsl`, `tool_missing`, `toast_failed` | `false` | `false` |
| `interop_failed` | `true` | `false` |
| `outcome_unknown` | `false` | `true` |

## Hooks

None in 0.x. wsl-notice shows one toast and takes no policy decisions; gating and routing belong to the caller. A `post-send` hook that notifies on failure could call this tool again and loop, which is the other reason there are none. Adding hooks later is additive.

## Calling from automation

Canonical call: `wsl-notice send --stdin-json --json --actor auto:<name> [--dedupe-key=<hex>]`, the payload on stdin, absent fields omitted (never null). A caller that has a key passes a hex digest of it with the `=` form, never the raw id; the key changes nothing but the log in 0.x. Decide by exit code:

| Exit | Meaning | Caller action |
|---|---|---|
| 0 | sent | Record it. |
| 22 | disabled, a successful no-op | Nothing. A shell caller writes `rc=$?; [ "$rc" = 22 ] && rc=0`. |
| 20 | the payload was refused | Do not retry; surface once per reason. After `link_invalid` or `link_scheme` a caller may retry once without `link`, putting the URL in the message. |
| 12 (false, false) | not sent; it will fail again until a person changes the config or the host | Do not retry; surface once per code. |
| 12 (true, false) | not sent, worth retrying (`interop_failed`) | Retry with the caller's own backoff and cap; the tool has already retried once inside its deadline. |
| 12 (false, true) | possibly shown (`outcome_unknown`) | Do not retry; record it as possibly shown. A retry can show a second toast. |
| 2, 1 | a usage or config fault, or an internal error | Do not retry; surface as a fault of the deployment. Exit 1 with no envelope: the outcome is unknown. |

The caller must prevent what refuses a payload: cut title and message to at most 100000 characters, always supply a non-empty message (copy the title into it if needed), use one of the four levels, and percent-encode links and give hosts as punycode. The caller's own kill timeout must be longer than the tool's timeout plus 5 s, because stopping the child can take 4 s; a caller that kills the tool gets no envelope and treats the outcome as unknown; SIGTERM is preferred, the tool treats it as the deadline. The tool may exit without reading all of stdin, so a caller ignores EPIPE and SIGPIPE on the stdin write and decides by the exit code alone.

The tool has no lock: a caller sends one toast at a time, because parallel PowerShell starts fail with `interop_failed`. To send to several destinations use one config file per destination with `--config`. `--config` and `--file` choose which files the tool reads and, through the config, the program that receives every notification; no rule on the command text can make that safe, because the environment of the process is trusted, so an agent that must not choose a destination does not call the tool: it calls the caller's router. The log depends on a usable state directory; a state problem only warns on stderr. A doctor runs `wsl-notice validate --json [--config <path>]`, reads `enabled`, and treats a failed `platform` or `powershell` check of a tool with `enabled` false as information. A cron or systemd caller runs `validate --probe` once from the same environment; `interop_failed` after a good probe is transient, so retry it with a cap of three.

## Config

Config is read from `.config/wsl-notice/config.yaml` under the user's home directory from the password database (`pwd.getpwuid(os.getuid()).pw_dir`); neither `HOME` nor `XDG_CONFIG_HOME` is read. `--config <path>` reads that file instead. Drop-ins are `config.d/*.yaml` in the directory that holds the config file, in lexical order; maps merge. A missing default config file means defaults; a missing file named by `--config` exits 2 `config_invalid` with `details.key` `--config`. Files must be at most 65536 bytes and regular files (symlinks are followed; the opened descriptor is checked), and are parsed with a strict YAML loader (a `yaml.SafeLoader` whose resolvers keep only `true` and `false` as booleans and only decimal integers and decimal floats as numbers, so `no`, `off`, `yes`, `1:30`, `0x10`, `1_0` and `.nan` stay strings and fail the type check); the top level must be a mapping. A file that cannot be read or parsed, a YAML error, `RecursionError` or `MemoryError` while loading, exits 2 `config_invalid` with `details.key` set to the file path and a fixed message: the YAML error text (it quotes the file) and every value are never printed. Merging and validation look only at known keys: an unknown key is refused before its value is examined. Booleans are only the YAML booleans `true` and `false`; a boolean is never accepted for a number, and a number must be finite. There are no environment overrides and no per-repo override: precedence is flags, then config, then defaults. Validation messages are built from the key and the rule only (Pydantic's input value is never used).

Durations (`timeout`) are a number of seconds, or a string matching `[0-9]+(\.[0-9]+)?(ms|s|m)?` (no unit means seconds); each key has its own default and range and no key constrains another. The tool enforces the range at load with a check that is false for NaN (`lo <= x <= hi`). Invalid config exits 2 `config_invalid`, `details.key` naming the key; a message names the key and the rule and never the value.

Schema: [schemas/config.json](../schemas/config.json); example: [config.example.yaml](../config.example.yaml). `level_prefix` values are run through the title rule (sanitizer steps 1 to 5) and must come out unchanged, else `config_invalid`; they are not silently rewritten.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | `false` makes `send` exit 22. |
| `powershell_bin` | `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe` | Absolute path of Windows PowerShell 5.1 (`^/[^\x00\\]{1,4096}$`: no drive letters, no backslashes, no `PATH` search). |
| `app_id` | the Windows PowerShell identity | Application identity the toast is shown under. |
| `level_prefix` | `{info: "", warn: "WARN:", action: "ACTION:", error: "ERROR:"}` | Title prefixes, at most 16 characters each; run through the title rule and unchanged by it. |
| `max_title` | `100` | Title limit, 20 to 256. |
| `max_body` | `1000` | Message limit, 100 to 5000. |
| `timeout` | `10s` | One monotonic deadline for a `send` ([Toast construction](#toast-construction)), 5s to 60s. |
| `link_hosts` | unset | List of 1 to 32 host names in lowercase, each of at least two labels `[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?` joined by single dots, no leading or trailing dot, at most 253 characters; an entry that does not match is `config_invalid` (`details.key: link_hosts`). When set, see the link rule under [Notification](#notification). |
| `log` | `true` | `false` writes no `log.jsonl`. |

## Environment variables

There are no `WSL_NOTICE_<KEY>` overrides, and no variable selects the config or the state directory (`HOME`, `XDG_CONFIG_HOME` and `XDG_STATE_HOME` are not read; both locations come from the password database). The tool itself reads `WSL_INTEROP` (the interop socket choice). It copies `PATH`, `HOME`, `LANG`, `LC_ALL` and `WSL_DISTRO_NAME` to the child as listed under [Toast construction](#toast-construction). WSL is detected from `/proc/sys/kernel/osrelease` only. It reads no credentials.

## Log

`log.jsonl` ([schemas/log.json](../schemas/log.json)) holds one record per outcome that reaches step 3 or later. Not logged: exit 1, exit 2, exit 22 and dry runs. A write takes an exclusive `flock` on `log.lock`, polled without blocking every 20 ms for at most 1 s (if the lock is not obtained, the record is skipped with the same warning as an unusable state directory), rotates when the file is 1 MiB or more (`log.jsonl.2` to `.3`, `.1` to `.2`, `log.jsonl` to `.1`), reopens `log.jsonl` after a rotation, and appends the record with one write of less than 1 KiB. The log step is outside the deadline, is bounded as stated, and a log write failure (including a full disk or a timeout) warns on stderr and never changes the exit code. Config `log` false writes nothing.

`outcome` is `sent` (exit 0), `refused` (exit 20), `outcome_unknown` (exit 12 whenever the envelope's `unknown_outcome` is true) or `failed` (any other exit 12). `code` is `refused` for exit 20, the error code for exit 12 and null on success; `reason` is the refusal reason for exit 20 and null otherwise. `actor_source` is `flag`, `tty` or `default`. `duration_ms` (integer, never null) is the time from the start of step 3 to just before the log write. `ts` is UTC, `YYYY-MM-DDTHH:MM:SS.mmmZ`. Nullable and always present: `level` (null when no valid level was read), `bytes`, `truncated`, `message_sha256` and `dedupe_key_sha256` (null for a refusal before rendering, or when not computed), and `attempts` (the number of PowerShell spawns; null when nothing was spawned). Records never contain a title, message, link, host or credential. There is no audit file: a notification changes nothing in a system of record.

Fields: `{ts, tool, version, outcome, exit, code, reason, level, bytes, truncated, duration_ms, attempts, message_sha256, dedupe_key_sha256, actor, actor_source}`; every field is always present. `message_sha256` and `dedupe_key_sha256` follow the hashing rule under [`send`](#send). `not_wsl` is logged (it is step 5). The file keeps 3 rotated files.

## Files

The state directory is `.local/state/wsl-notice` under the home directory from the password database; `XDG_STATE_HOME` is not read. It is created with mode 0700, opened with `O_DIRECTORY` and `O_NOFOLLOW`, and checked on that descriptor to be owned by the user with no group or other permissions. Every file below it is opened relative to that descriptor (`dir_fd`) with `O_NOFOLLOW`, `O_NONBLOCK` and `O_CLOEXEC` (the log also with `O_APPEND` and mode 0600) and must be a regular file on `fstat`; a FIFO, a symlink or a device makes the state unusable instead of blocking the tool. It is used only when it is needed (a log record). If it cannot be used the tool warns once on stderr, writes no log record, and the exit code is unchanged: a state problem never blocks or fails a send.

| Path | Content |
|---|---|
| `.config/wsl-notice/config.yaml`, `config.d/*.yaml` (under the home directory) | Config. |
| `<state directory>/log.jsonl` (and `.1` to `.3`) | Log records. |
| `<state directory>/log.lock` | Lock for log writes and rotation. |

There is no database and no migration, and no file is written to the Windows side.

## Labels

wsl-notice creates no long-lived objects and takes no labels.

## TUI

None in v1.

## Residual risks

- **No display confirmation.** `sent` means PowerShell returned success: the call completes with exit 0 and the toast is added to the notification history (spike), but on-screen display under the default identity is not yet confirmed (the user saw no toast in the spike); 0.1.0 is gated on a person seeing one. An unregistered `app_id`, Focus Assist, do-not-disturb or notification settings can suppress the toast with no error ([design.md](design.md), Live verification).
- **Start cost and interop flakiness.** Each call starts a Windows process (about one second) and can fail with an interop error after a hung or killed call, or when a long-lived tmux or cron job holds a stale `WSL_INTEROP` socket; one automatic retry and the socket fallback cover the cases seen so far (unverified), and the tool trusts only a root-owned socket as the interop transport. A cron job's interop works only when a socket exists and accepts (design.md, V14); run `validate --probe` from the cron line once.
- **The script is trusted, the text is not.** The only defense for the text is that it reaches PowerShell on stdin and becomes XML text nodes; a change to the constant script that concatenated strings into XML would reopen image, audio and action injection (a golden test pins the script).
- **The environment is trusted.** Interpreter and loader variables (`PYTHONPATH`, `LD_PRELOAD`, `OPENSSL_CONF`) run code in the tool; the tool reads no variable that selects its config or its program (there are no `WSL_NOTICE_<KEY>` overrides and the config path comes from the password database), but a caller that can set the environment of the process is inside the trust boundary.
- **The file at `powershell_bin` receives every notification and is trusted.** The path comes from the user's config, not from `PATH`, and `--config` can name it.
- **`--config` selects the destination.** The program that receives every notification is named by the config file; no rule on the command text prevents it.
- **Links open in the default handler.** `link` is limited to `http` and `https`; the toast API itself accepts any protocol. With `link_hosts` unset a producer can aim the tap target at any web address; the toast does not show the URL. Whether a tap opens the link has never been tried (unverified).
- **Same-user visibility.** Another Windows or WSL process of the same user can read the notification from the Windows notification history; callers must not send secrets.
- **`level` is advisory.** The prefix does not authenticate the sender: a producer can write `ERROR:` in a title.
- **Same-user threat boundary.** A process that can write the user's config or `config.d` can redirect the tool; the state directory is checked but the config is trusted.
- **Short hash.** `message_sha256` is a 64-bit unsalted hash; a short message can be confirmed against the log by a reader of the 0600 file.
- **No concurrency limit.** The tool has no lock; callers send one toast at a time. A caller that retries after `outcome_unknown` may show two toasts.
- **Not supported:** native Windows, Linux desktop and macOS notifiers (separate tools), PowerShell 7, WSL1 (not tested), custom icons, sounds, buttons, images and progress bars.

## Platform notes

WSL2 on Windows 10 or 11 with Windows PowerShell 5.1 reachable through interop (`[interop] enabled=true` in `/etc/wsl.conf`) is the supported platform; the default `powershell_bin` assumes the default automount root. Outside WSL the tool exits 12 `not_wsl` (a dry run still works).
