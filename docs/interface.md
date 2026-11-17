# wsl-notice: interface

Status: 0.1.0-design, 2026-11-17. Written before implementation (architecture section 18.1). This document is the single normative source for wsl-notice's rules; `docs/design.md`, the README and the architecture link to its headings and do not restate them. The first release (0.1.0) is gated on a person seeing a toast under the default identity ([plan.md](plan.md), stage 0).

wsl-notice shows one Windows toast notification from a Linux process running inside WSL, through Windows PowerShell. It is the WSL member of a planned family of platform notifiers: a native Windows tool, other Linux desktops and macOS would be separate tools with their own names (`win-notice`, `linux-notice`, `mac-notice`). It is stateless: it keeps no database, only `log.jsonl`. It runs no hooks and takes no policy decisions: when to send, what to send and how often belong to its caller.

References: architecture (`../awkit-docs/architecture.md`) sections 5.1, 13.2, 15 and 20; design guide (`../awkit-docs/design-guide.md`).

## Concepts

### Notification

A notification is a JSON object ([schemas/notification.json](../schemas/notification.json)):

| Field | Type | Required | Rule |
|---|---|---|---|
| `title` | string | no | One line (the sanitizer turns newlines into spaces); cut at `max_title` characters after sanitizing ([Rendering](#rendering)). At most 100000 characters of input (else exit 20, reason `invalid_payload`). |
| `message` | string | yes | At most 100000 characters of input (else exit 20, reason `invalid_payload`). Must not be empty after sanitizing (else exit 20, reason `empty_message`). |
| `level` | `info`, `warn`, `action`, `error` | no | Default `info`. Ordered `info < warn < action < error`; the order matters to callers, not to wsl-notice. |
| `link` | string | no | 1 to 1024 printable ASCII characters, `http` or `https`; the rule is the **Link rule** below. |

**Input.** This paragraph is the input rule for every source.

```text
SHARED-3 (Input)
Exactly one source is used: JSON on stdin (send --stdin-json, also the default when stdin is not a terminal and no text flag is given), JSON from --file <path> (--file - is stdin, the same source), or flags. A JSON source together with --title, --message, --level or --link, or --stdin-json together with --file <path>, exits 2 usage. Flags alone need --message; stdin that is a terminal with no source and no --message exits 2 usage. A --file that does not exist, cannot be read or is not a regular file (the tool opens it with O_NONBLOCK and checks the opened descriptor; a symlink to a regular file is fine) exits 2 usage.
At most 1048576 bytes are read from a source. More, input that is not valid UTF-8 or that begins with a byte order mark, JSON that fails to parse, nests too deeply (RecursionError), repeats an object key or contains NaN or Infinity, a top-level value that is not an object, or input that does not arrive within the deadline (SHARED-4) exits 20 refused, reason invalid_payload, details.pointer "".
Fields: an unknown field, a field of the wrong type (null is a wrong type: absent fields are omitted), a level outside info, warn, action, error, or a title or message of more than 100000 characters exits 20 invalid_payload with details.pointer set to "/<field>". A flag value is checked like the same JSON field and uses the same pointer. A message that is empty after sanitizing is empty_message (the schema has no minLength for it). Error messages and details never contain an input value, only the pointer and the rule.
Flag and combination errors (an unknown flag, a missing value, a bad --actor) exit 2 usage.
```

Flag text is untrusted and goes through the same [sanitizer](#sanitizer) as stdin text; flags exist for humans. The deadline is described under [Toast construction](#toast-construction).

**Link rule.**

```text
SHARED-2 (Link rule)
link is optional and is checked in this order. (1) It is a string of 1 to 1024 printable ASCII characters (U+0021 to U+007E); anything else, including whitespace, control characters, non-ASCII characters and an empty string, is reason link_invalid (callers percent-encode paths and give hosts as punycode). (2) If it starts with a scheme token ([A-Za-z][A-Za-z0-9+.-]*:) whose lowercase form is not http or https, the reason is link_scheme. (3) It must match (?i:https?)://(?:[A-Za-z0-9-]+(?:\.[A-Za-z0-9-]+)*|\[[0-9A-Fa-f:.]+\])(?::[0-9]{1,5})?(?:[/?#][\x21-\x7e]*)? as a whole, and a port must be 1 to 65535, else link_invalid (no userinfo, no empty host, no backslash between host and path). The scheme is lowercased wherever the link is rendered. The schema pattern is a coarse gate for the same rule; the tool is authoritative. A link names a place to open and is never a promise that it is safe: a caller that builds links from untrusted text takes that risk.
```

When config `link_hosts` is set, the link's host (lowercased, without port) must also equal an entry or end with `.` plus an entry, else exit 20 `link_invalid` (`details.pointer: "/link"`). With `link_hosts` unset a producer can aim the tap target at any web address; the toast does not show the URL.

### Sanitizer

```text
SHARED-1 (Sanitizer)
Applied to title and message before rendering, in this order. Every pattern in this document is matched against the whole string (re.fullmatch); a trailing newline never passes.
1. Input is read as bytes and decoded as strict UTF-8, never through the locale. Invalid UTF-8 or a lone surrogate: exit 20 refused, reason invalid_payload.
2. CRLF, CR, U+0085, U+2028 and U+2029 become LF. The tab becomes a space.
3. These characters are removed: every character of category Cc other than LF, and of categories Cf, Cs and Co; the noncharacters U+FDD0 to U+FDEF and U+FFFE and U+FFFF of every plane; and the blank-looking fillers U+034F, U+115F, U+1160, U+17B4, U+17B5, U+180B to U+180D, U+2800, U+3164, U+FE00 to U+FE0F, U+FFA0 and U+E0100 to U+E01EF. This removes zero-width, direction and tag characters, the soft hyphen and the byte order mark. It also removes the zero-width joiner and variation selector 16, so some emoji sequences are shown as separate or text-style emoji.
4. Trailing whitespace of every line is removed (str.rstrip), then runs of more than two LF become two, then leading and trailing whitespace is removed (str.strip, Unicode whitespace).
5. title only: every LF becomes a space, runs of spaces become one space, and the result is stripped again.
A message that is empty after step 4 is refused with reason empty_message. A title that is empty after step 5 is absent.
```

No markup escaping is done in the tool, because the text never becomes markup: the toast is built from XML text nodes on the Windows side ([Toast construction](#toast-construction)). As a second line of defense the constant script removes `[\x00-\x08\x0B\x0C\x0E-\x1F￾￿]` before every `InnerText` assignment (U+FFFE and U+FFFF break `InnerText`).

### Rendering

- The toast title is the level's entry in `level_prefix` (defaults: none for `info`, `WARN:`, `ACTION:`, `ERROR:`; one-line strings up to 16 characters) joined with a space to the title; with neither, the toast has the message as its only text line.
- The title is limited to `max_title` characters (default 100, 20 to 256) and the message to `max_body` (default 1000, 100 to 5000), in Unicode code points. The limits sit below what Windows is known to accept because on-screen truncation is not observable. The rendered title is at most 273 characters.
- `link`, when given, is the toast's tap action ([Toast construction](#toast-construction)); its scheme is lowercased.

```text
SHARED-13 (Cuts)
Limits count Unicode code points. A title longer than its limit L is cut to its first L-1 characters plus … (L in all, before the level prefix is added) and truncated is true. The prefix is added after the cut and does not count. A message cut for length keeps as many characters as fit, has trailing whitespace removed, and ends with the marker so that the result, marker included, is within the limit; truncated is true. A cut may split a combining or emoji sequence; that is accepted. When the title is empty and the prefix is not, the title line is the prefix alone with no trailing space; when both are empty the title is absent.
```

In wsl-notice the marker is `…` only (a small surface, intentionally different from slack-notice and ha-notice) for both `max_title` and `max_body`. `message_sha256` and `bytes` are defined under [Log](#log) and [Output](#output).

## Commands

### `send`

`wsl-notice send [--stdin-json | --file <path|->] [--title <t>] [--message <m>] [--level <l>] [--link <url>] [--dedupe-key <k>]`

Order, with the exit code of each refusal:

1. Parse flags (exit 2 `usage`).
2. Load config (exit 2 `config_invalid`). `enabled: false` prints `{"outcome":"disabled"}` (with `--json`) and exits 22 here, before input is read.
3. Read and validate input; the deadline starts here ([Toast construction](#toast-construction)). Exit 2 `usage` for a conflicting source; exit 20 with reason `invalid_payload`, `empty_message`, `link_invalid` or `link_scheme`.
4. Render ([Rendering](#rendering)). With `--dry-run`, print the result with `rendered` (the JSON handed to PowerShell) and exit 0: steps 5 and 6 are skipped (so a dry run succeeds on any platform), nothing is started and nothing is logged.
5. Check the platform: not WSL exits 12 `not_wsl`; then the binary: `powershell_bin` missing or not executable exits 12 `tool_missing` (`details.install`).
6. Run PowerShell ([Toast construction](#toast-construction)) and map the result to an exit code ([Exit codes](#exit-codes)).
7. Append the [log record](#log) and print the result.

```text
SHARED-9 (Hashing, serialization, keys, actor)
Every JSON document that the tool sends, hashes or measures is serialized with json.dumps(obj, ensure_ascii=False, separators=(",", ":")), keys in the order given in this document, and encoded as UTF-8. bytes in the output and the log is the length of the request payload in that form (slack-notice and ha-notice: the request body; wsl-notice: the JSON handed to PowerShell, before base64).
message_sha256 is the first 16 hex characters of the SHA-256 of the UTF-8 bytes of the sanitized title (empty string when absent), a LF, and the sanitized message, before any cut, prefix or link, so the same input gives the same value in every notifier. dedupe_key_sha256 is the first 16 hex characters of the SHA-256 of the key's UTF-8 bytes. Both are null when not computed (a refusal before the text was sanitized, or no key).
--dedupe-key takes any non-empty string of at most 1024 characters (valid UTF-8, else exit 2 usage). It is only hashed; the send output echoes the caller's own key or null. --decision-id (repeatable) is accepted for uniformity with the design guide and ignored: the log is hash-only and there is no audit file.
--actor must match ^(human|auto):[A-Za-z0-9._@/-]{1,64}$ (else exit 2 usage). Without it the actor is human:<user> when stdin is a terminal (actor_source tty) and auto:unknown otherwise (default), where <user> is the login name from the password database with every character outside [A-Za-z0-9._@-] replaced by _ and cut to 64 characters; the design guide's me.name does not exist here because the tool has no me key.
```

In wsl-notice the key order of the JSON handed to PowerShell is `title`, `body`, `launch`, `app_id`. The dedupe key changes nothing but the log in 0.x: a retry after `outcome_unknown` can show a second toast.

### `validate`

```text
SHARED-10 (validate)
<tool> validate [--probe] [--test-message]  (slack-notice: validate [--test-message])
validate never prompts. It prints {ok, enabled, checks: [{name, ok, message}]} (schemas/cli/validate.json), where enabled is the configured value. Checks run in the order listed for the tool; a failed check ends the run and later checks are omitted, so ok is true only when every printed check is ok. It exits 0 when ok is true, otherwise with the exit code of the failing check (config: 2; credential, platform, powershell, probe and test_message: the exit code of the corresponding failure, 12 or 20). When the config cannot be loaded it prints the error envelope instead and exits 2. Without --json it prints one line per check on stderr.
--test-message sends one notification through the normal render and send path (level info, title "<tool>", message "validate: test message"; no dedupe key or tag) and adds the check test_message; its message says that delivery is not observable and tells the user to look at the destination. It is a send: it is logged like a send, and it exits 22 like send when the tool is disabled. --probe makes only the calls that send no notification. Automation does not call --test-message.
```

For wsl-notice the checks, in order, are `config`, `platform` (`not_wsl`, exit 12), `powershell` (`tool_missing`, exit 12; its message names the resolved `powershell_bin`), `probe` and `test_message`. `--probe` runs the constant probe script ([Constant script](#constant-script)) once: it proves interop works and measures start time, and adds the top-level fields `powershell_version` and `start_ms`; the probe check message says which interop socket source was used (inherited or fallback). The test message has title `wsl-notice` and message `validate: test message`, and its check message says "look at your screen". With `--probe` or `--test-message` and `enabled: false`, validate makes no call and exits 22 with `{"outcome":"disabled"}`.

## Global flags

| Flag | Meaning |
|---|---|
| `--json` | Machine output on stdout ([Output](#output)). |
| `--config <path>` | Config file instead of `~/.config/wsl-notice/config.yaml` (drop-ins are read next to it). |
| `--dry-run` | `send` only: render and print, start nothing, write no log record (`attempts` is null). |
| `--actor <actor>` | Recorded in the log; grammar in the hashing rule under [`send`](#send). |
| `--decision-id <id>` | Accepted and ignored (repeatable). |
| `--verbose`, `--quiet` | More or less human output on stderr. |
| `--version` | Print `wsl-notice <version>`. |

## Toast construction

wsl-notice starts Windows PowerShell with an argument list that never contains notification text and is the same on every call:

`<powershell_bin> -NoProfile -NonInteractive -EncodedCommand <base64 of the constant script>`

- `powershell_bin` is an absolute path (config; default `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe`). There is no `PATH` search and no environment override: a user whose automount root differs sets it in config. Windows PowerShell 5.1 is the supported host; PowerShell 7 is not needed and not used.
- WSL is detected only by `/proc/sys/kernel/osrelease` containing `microsoft` (case-insensitive), read through the gateway; `WSL_DISTRO_NAME` is ignored. WSL1 passes the check and is not distinguished (unverified).
- The notification travels on stdin as one line: the base64 of a UTF-8 JSON object `{title, body, launch, app_id}` (`title` and `launch` null when absent), followed by a newline. Raw UTF-8 on stdin is not used because the console's input code page corrupts it. `app_id` is only ever passed inside that JSON, never in a command line.
- The child's environment is built, not inherited: only `PATH`, `HOME`, `LANG`, `LC_ALL`, `WSL_DISTRO_NAME` and `WSL_INTEROP` are copied, and only when set; `WSLENV` is not passed. The working directory is `/`. No notification text is placed in the environment or the argument list.
- Before every spawn the tool checks that `WSL_INTEROP` names a socket that accepts a connection. If it does not (unset, or a stale value held by a long-lived tmux or a cron job), it takes the newest `/run/WSL/*_interop` socket that accepts a connection and passes that value to the child only; if none works, the result is `interop_failed` with a message that says the interop socket is stale. This is best effort (design.md, V14).
- `link`: when present and valid, the toast gets `launch` set to it with `activationType="protocol"`. Windows accepts any scheme there, which is why the tool allowlists.
- `app_id` is the application identity the toast is shown under (config `app_id`, default the Windows PowerShell identity, `{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe`, which needs no registration; the toast should show as coming from "Windows PowerShell"). It matches `^[A-Za-z0-9_.{}\\ -]{1,256}$`. An identity that is not registered gives no error and may show nothing.
- On failure the script writes one fixed token to stderr and exits 3: `decode`, `json`, `xml` or `show`, naming the step that failed, and writes nothing else. `details.stage` is that token when stderr, trimmed, is exactly one of them, and `unknown` when the exit was 3 with any other stderr. Other stderr text is discarded (never printed or logged). Stderr is read up to 4 KiB.

**Deadline.**

```text
SHARED-4 (Deadline)
Config timeout is one wall-clock deadline for the whole send, counted from the start of input reading (step 3) to the end of step 6. It covers reading input, waiting for the send lock, name resolution, connecting, TLS, writing the request, reading the response, retry waits, starting the child process and waiting for it. The network or child phase runs in a worker thread; the main thread waits for it with the time that remains and, at the deadline, reports the result below, writes the log record, prints the output and exits without waiting for the worker. A first SIGINT or SIGTERM is treated as the deadline having passed.
The result at the deadline depends on how far the request got. Before the first byte of the request was written (or before the child was started): slack and ha exit 12 network_error (retryable true, details.timeout true), or unavailable when waiting for the send lock. After the first byte was written or the child was started: exit 12 outcome_unknown. Input that has not arrived at the deadline is invalid_payload (SHARED-3). A wait (a retry wait or the lock wait) that is longer than the time that remains ends the send at once with the code of the last result, without sleeping.
```

For wsl-notice there is no send lock, no name resolution and no request, so the first paragraph covers reading input, the interop socket check, starting the child and waiting for it. After the child was started, the deadline result is `outcome_unknown` (`details.attempts`); a deadline before the child was started is `interop_failed`. At the deadline wsl-notice sends SIGTERM to the child's process group (which it created), waits 2 s, sends SIGKILL to the group, waits at most 2 s more and reports `outcome_unknown` whether or not the child has gone (the output then says the child may still run); this stop sequence can take up to 4 s beyond the deadline. A SIGINT or SIGTERM received by wsl-notice does the same before it exits. A BrokenPipeError while writing the stdin line is not an error by itself: the exit status of the child decides. `timeout` is 5s to 60s (default 10s): a cold start takes 1 to 5 s and the interop retry needs room.

**Interop retry.** An exit 1 whose captured stderr (cap 4 KiB, case-sensitive substring) contains `UtilAcceptVsock` is retried once after 500 ms, inside the same deadline and only when at least 2 s remain. The second attempt is classified by the table below; `UtilAcceptVsock` again is `interop_failed`. `attempts` is the number of PowerShell spawns made.

| PowerShell result | Outcome |
|---|---|
| Exit 0 | Exit 0, `sent` (the toast was handed to Windows). |
| Exit 3 | Exit 12 `toast_failed`, `details.stage`, `details.attempts`. |
| Exit 1 with `UtilAcceptVsock` | One retry (above); a second failure is exit 12 `interop_failed`. |
| Any other exit, or the script never started (no stage token) | Exit 12 `interop_failed`. |
| Spawn error `ENOENT`, `EACCES`, `ENOTDIR` or `ENOEXEC` | Exit 12 `tool_missing` (`details.install` covers the path and interop). Any other spawn error is exit 1. |
| No `WSL_INTEROP` socket accepts a connection | Exit 12 `interop_failed` (stale socket). |
| The deadline passed after the spawn | Exit 12 `outcome_unknown`. |

`retryable` and `unknown_outcome` per code are in the [Error envelope](#error-envelope). Whether the toast was displayed, dismissed or suppressed (Focus Assist, do-not-disturb) is not observable, so output says `sent`, never `displayed`.

### Constant script

The script text is constant, kept in the package, encoded into `-EncodedCommand` and pinned by a golden test (the listing, the encoded argument and the XML skeleton). It is specified here:

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
  $strip = '[\x00-\x08\x0B\x0C\x0E-\x1F￾￿]'
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

Rules the listing must keep, whatever its formatting: one line is read from stdin and an empty line is stage `decode`; text reaches the XML only through `InnerText`, and `launch` only through `SetAttribute` on the root `toast` element; there are no `audio`, `actions`, `image` or `duration` elements; every failure writes its stage token to stderr and exits 3, and nothing else is written. The probe script used by `validate --probe` is also constant: it prints `$PSVersionTable.PSVersion.ToString()` and exits 0, and the tool accepts only output matching `[0-9]+(\.[0-9]+){1,3}`. A second golden test shows the XML produced for injection payloads (images, audio, actions, a `launch` with quotes) and asserts only text nodes and one attribute.

## Output

With `--json`, `send` prints one object ([schemas/cli/send.json](../schemas/cli/send.json)): `{outcome, tool, level, truncated, bytes, duration_ms, attempts, dedupe_key, rendered}`. `outcome` is `sent`, `dry_run` or `disabled`; `bytes` is the size of the JSON handed to PowerShell before base64; `attempts` is 1 or 2 for `sent` and null for a dry run; `rendered` is present only for a dry run. A disabled tool prints `{"outcome":"disabled"}` and nothing else.

```text
SHARED-12 (Output)
Global flags may appear before or after the subcommand. The tool looks for --json in argv before it parses anything, so a usage error with --json prints the envelope (code usage) on stdout; --help and --version are exempt. Errors always print one line on stderr; with --json they also print the error envelope on stdout. With --json a successful send prints one object on stdout; without it human text goes to stderr and stdout is empty (a dry run prints the rendered text on stderr). A disabled tool prints {"outcome":"disabled"} on stdout with --json and one line on stderr without it, and exits 22 either way. stderr is advisory (warnings, verbose lines); the contract is the exit code plus the stdout JSON.
```

## Exit codes

```text
SHARED-6 (Exit codes and envelope)
Codes 0 to 21 follow design guide section 4.2. Exit 0: sent or a dry run. Exit 1: internal error. Exit 2: usage or invalid configuration (codes usage, config_invalid). Exit 12: failed for a reportable reason (the codes of the tool, below). Exit 20: refused before sending because of the payload; code refused with a required reason. Exit 22: nothing sent because the tool is disabled in config (not an error; output {"outcome":"disabled"}). Exits 10, 11, 13 and 21 are never used: the tool stores no objects, has no hooks and never prompts. A caller treats exit 22 as a successful no-op.
```

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

```text
The error envelope is {"error": {"code", "message", "reason", "details", "retryable", "retry_after_s", "unknown_outcome"}}; only code and message are required and the other keys are present with null when they do not apply. reason is set only on exit 20. retry_after_s is set only for rate_limited (the last wait performed, or the wait that did not fit the deadline). retryable means that sending the same notification again is safe. unknown_outcome means that the notification may already have been delivered. They are set as follows.
- Exit 2 and exit 20: retryable null, unknown_outcome null.
- Exit 1: retryable false; unknown_outcome true when the failure happened after the first byte of the request was written or after the child was started, else false. The envelope on exit 1 is best effort: exit 1 with no envelope means the outcome is unknown.
- Exit 12: both are booleans (below).
details by code or reason: config_invalid: key (a config key, or the file path for a file that cannot be read or parsed); reason invalid_payload: pointer; toast_failed: stage, attempts; interop_failed and outcome_unknown: attempts. A message never contains an input value or a credential.
```

The envelope is [schemas/cli/error.json](../schemas/cli/error.json). `retry_after_s` is always null in wsl-notice (the key stays for a uniform shape). Exit 12 booleans for wsl-notice: not sent and permanent (`retryable` false, `unknown_outcome` false): `not_wsl`, `tool_missing`, `toast_failed`; not sent and worth retrying (true, false): `interop_failed`; possibly shown (false, true): `outcome_unknown`. Further details: `tool_missing` carries `install` (fixed text naming the `powershell_bin` key and the WSL `[interop] enabled` and automount settings), `link_invalid` from `link_hosts` carries `pointer` `/link`. `details.attempts` is the number of PowerShell spawns made (1 or 2); it is null for `interop_failed` when no child was started.

| Exit 12 code | `retryable` | `unknown_outcome` |
|---|---|---|
| `not_wsl`, `tool_missing`, `toast_failed` | `false` | `false` |
| `interop_failed` | `true` | `false` |
| `outcome_unknown` | `false` | `true` |

## Hooks

None in 0.x. wsl-notice shows one toast and takes no policy decisions; gating and routing belong to the caller. A `post-send` hook that notifies on failure could call this tool again and loop, which is the other reason there are none. Adding hooks later is additive.

## Calling from automation

```text
SHARED-14 (Calling from automation)
Canonical call: <tool> send --stdin-json --json --actor auto:<name>, the payload on stdin, absent fields omitted (never null). Decide by exit code: 0 sent; 22 disabled, a successful no-op; 20 the payload was refused, do not retry (a caller may retry once without link after link_invalid or link_scheme, putting the URL in the message); 12 read retryable and unknown_outcome: retry only when retryable is true, and when unknown_outcome is true the notification may already have been delivered; 2 a usage or config fault of the caller or the deployment; 1 an internal error (no envelope means the outcome is unknown). The tool exits before reading stdin when it is disabled or its config is invalid, so a caller ignores EPIPE or SIGPIPE on the stdin write and decides by the exit code alone. The tool does no queueing: a caller sends one notification at a time per tool and does not start several in parallel. To send to several destinations use one config file per destination with --config. A permission profile for an agent allows only send --stdin-json --json --actor ... and denies --config, --file and the text flags, because those choose the destination and read arbitrary files.
```

wsl-notice has no send lock: a toast takes about one second, and a flush of many in parallel is the caller's fault. `--dedupe-key` has no effect on the toast, so a caller that retries after `outcome_unknown` may show a second toast; wsl-notice therefore reports `outcome_unknown` as not retryable.

## Config

```text
SHARED-8 (Config)
Config is read from ~/.config/<tool>/config.yaml (XDG_CONFIG_HOME is honoured when it is an absolute path), then config.d/*.yaml in lexical order; maps merge. A missing default config file means defaults; a missing file named by --config exits 2 config_invalid with details.key "--config". Files are parsed with yaml.safe_load, must be at most 65536 bytes and regular files (symlinks are followed; the opened descriptor is checked), and the top level must be a mapping; a file that cannot be read or parsed exits 2 config_invalid with details.key set to its path. Unknown keys are refused. Booleans are only the YAML booleans true and false; a boolean is never accepted for a number, and a number must be finite. There are no environment overrides and no per-repo override: precedence is flags, then config, then defaults. The environment variable that holds a credential is named by the config key shown in the table; it is the only variable the tool reads for a secret.
Durations (timeout and others) are a number of seconds, or a string matching [0-9]+(\.[0-9]+)?(ms|s|m)? (no unit means seconds); the tool enforces the range of each key at load with a check that is false for NaN (lo <= x <= hi). Invalid config exits 2 config_invalid, details.key naming the key; a message names the key and the rule and never the value (for a name key that looks like a URL it says "must be a variable name").
```

wsl-notice has no credential, so the last sentence of the first paragraph does not apply. Schema: [schemas/config.json](../schemas/config.json); example: [config.example.yaml](../config.example.yaml). `level_prefix` values are run through the sanitizer rule (one line) at load; a value that the sanitizer would change is `config_invalid`, not silently rewritten.

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `true` | `false` makes `send` exit 22. |
| `powershell_bin` | `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe` | Absolute path of Windows PowerShell 5.1 (`^/[^\x00\\]{1,4096}$`: no drive letters, no backslashes, no `PATH` search). |
| `app_id` | the Windows PowerShell identity | Application identity the toast is shown under. |
| `level_prefix` | `{info: "", warn: "WARN:", action: "ACTION:", error: "ERROR:"}` | Title prefixes, at most 16 characters each. |
| `max_title` | `100` | Title limit, 20 to 256. |
| `max_body` | `1000` | Message limit, 100 to 5000. |
| `timeout` | `10s` | One wall-clock deadline for a `send` ([Toast construction](#toast-construction)), 5s to 60s. |
| `link_hosts` | unset | Optional list of 1 to 32 lowercase host names (`[a-z0-9.-]{1,253}`); when set, a link's host must equal an entry or end with `.` plus an entry. |
| `log` | `true` | `false` writes no `log.jsonl`. |

## Environment variables

There are no `WSL_NOTICE_<KEY>` overrides. The tool reads `XDG_CONFIG_HOME`, `XDG_STATE_HOME`, `HOME` and `WSL_INTEROP` (the interop socket check) and copies a fixed list of variables to the child ([Toast construction](#toast-construction)). WSL is detected from `/proc/sys/kernel/osrelease` only. It reads no credentials.

## Log

```text
SHARED-11 (State and log), second paragraph
log.jsonl holds one record per outcome that reaches step 3 or later. Not logged: exit 1, exit 2, exit 22 and dry runs. A write takes an exclusive flock on log.lock for the few microseconds it needs, rotates when the file is 1 MiB or more (log.jsonl.2 to .3, .1 to .2, log.jsonl to .1), and appends the record with one write of less than 1 KiB. A log write failure (including a full disk) warns on stderr and never changes the exit code. config log false writes nothing.
outcome is sent (exit 0), refused (exit 20), outcome_unknown (exit 12 whenever the envelope's unknown_outcome is true) or failed (any other exit 12). code is refused for exit 20, the error code for exit 12 and null on success; reason is the refusal reason for exit 20 and null otherwise. actor_source is flag, tty or default. duration_ms (integer, never null) is the time from the start of step 3 to just before the log write. Nullable and always present: level (null when no valid level was read), bytes, truncated and message_sha256 and dedupe_key_sha256 (null for a refusal before rendering, or when not computed), http_status (null unless a response arrived), attempts (wsl-notice; null when nothing was spawned). Records never contain a title, message, link, URL, base URL, service, host or credential. There is no audit file: a notification changes nothing in a system of record.
```

`~/.local/state/wsl-notice/log.jsonl` ([schemas/log.json](../schemas/log.json)). Fields: `{ts, tool, version, outcome, exit, code, reason, level, bytes, truncated, duration_ms, attempts, message_sha256, dedupe_key_sha256, actor, actor_source}`; every field is always present (`http_status` does not exist in wsl-notice). `message_sha256` and `dedupe_key_sha256` follow the hashing rule under [`send`](#send). `not_wsl` is logged (it is step 5). The file keeps 3 rotated files.

## Files

```text
SHARED-11 (State and log), first paragraph
The state directory is $XDG_STATE_HOME/<tool> when XDG_STATE_HOME is set to an absolute path, else $HOME/.local/state/<tool> (HOME from the password database when unset). It is created with mode 0700, opened with O_DIRECTORY and O_NOFOLLOW, and checked on that descriptor to be owned by the user with no group or other permissions; every file below it is opened relative to that descriptor (dir_fd), the log with O_APPEND and mode 0600. It is used only when it is needed (a log record or the send lock). If it cannot be used the tool warns once on stderr, writes no log record and takes no lock, and the exit code is unchanged: a state problem never blocks or fails a send.
```

| Path | Content |
|---|---|
| `~/.config/wsl-notice/config.yaml`, `config.d/*.yaml` | Config. |
| `~/.local/state/wsl-notice/log.jsonl` (and `.1` to `.3`) | Log records. |
| `~/.local/state/wsl-notice/log.lock` | Lock for log writes and rotation. |

wsl-notice takes no send lock. There is no database and no migration, and no file is written to the Windows side.

## Labels

wsl-notice creates no long-lived objects and takes no labels.

## TUI

None in v1.

## Residual risks

- **No display confirmation.** `sent` means PowerShell returned success: the call completes with exit 0 and the toast is added to the notification history (spike), but on-screen display under the default identity is not yet confirmed (the user saw no toast in the spike); 0.1.0 is gated on a person seeing one. An unregistered `app_id`, Focus Assist, do-not-disturb or notification settings can suppress the toast with no error ([design.md](design.md), Live verification).
- **Start cost and interop flakiness.** Each call starts a Windows process (about one second) and can fail with an interop error after a hung or killed call, or when a long-lived tmux or cron job holds a stale `WSL_INTEROP` socket; one automatic retry and the socket fallback cover the cases seen so far (unverified).
- **The script is trusted, the text is not.** The only defense for the text is that it reaches PowerShell on stdin and becomes XML text nodes; a change to the constant script that concatenated strings into XML would reopen image, audio and action injection (a golden test pins the script).
- **The file at `powershell_bin` receives every notification and is trusted.** The path comes from the user's config, not from `PATH`.
- **Links open in the default handler.** `link` is limited to `http` and `https`; the toast API itself accepts any protocol. With `link_hosts` unset a producer can aim the tap target at any web address; the toast does not show the URL. Whether a tap opens the link has never been tried (unverified).
- **Same-user visibility.** Another Windows or WSL process of the same user can read the notification from the Windows notification history; callers must not send secrets.
- **`level` is advisory.** The prefix does not authenticate the sender: a producer can write `ERROR:` in a title.
- **Same-user threat boundary.** A process that can write the user's config or set an XDG path can redirect the tool; the state directory is checked but the config is trusted.
- **Short hash.** `message_sha256` is a 64-bit unsalted hash; a short message can be confirmed against the log by a reader of the 0600 file.
- **No concurrency limit.** The tool has no lock; callers send one toast at a time.
- **Not supported:** native Windows, Linux desktop and macOS notifiers (separate tools), PowerShell 7, WSL1 (unverified), custom icons, sounds, buttons, images and progress bars.

## Platform notes

WSL2 on Windows 10 or 11 with Windows PowerShell 5.1 reachable through interop (`[interop] enabled=true` in `/etc/wsl.conf`) is the supported platform; the default `powershell_bin` assumes the default automount root. Outside WSL the tool exits 12 `not_wsl` (a dry run still works).
