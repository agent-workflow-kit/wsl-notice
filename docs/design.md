# wsl-notice: design

Status: 0.1.0-design, 2026-11-17. The rules are in [interface.md](interface.md), the single normative source; this document records decisions, modules and tests and links to interface headings without restating them. Architecture: `../awkit-docs/architecture.md` (v1.22).

## Role and layer

L1 primitive: show one Windows toast from a process running inside WSL. Works fully standalone. It is one of three sibling channel notifiers with the same contract (`slack-notice`, `ha-notice`), each designed to be called by a routing tool above it without knowing that tool exists. It was seeded as `desktop-notice`; the user renamed it so the name says what it is, and future platform notifiers (native Windows, other Linux desktops, macOS) are separate tools: `win-notice`, `linux-notice`, `mac-notice`.

## Uses

| Dependency | Layer | Binding | Notes |
|---|---|---|---|
| Windows PowerShell 5.1 through WSL interop | L0 third party | none (not an edge of the tool graph) | One short-lived process per notification; see [Toast construction](interface.md#toast-construction). |
| Python standard library (`subprocess` through `gateway.py`) | L0 | none | No third-party runtime dependency beyond Pydantic. |
| Pydantic (config validation) | library | none | Required by the design guide for config. The scaffold's `pyproject.toml` does not list it yet; adding it is part of stage 2 of [plan.md](plan.md). |

No tool dependencies: wsl-notice composes nothing.

## Decisions

| ID | Decision | Rationale |
|---|---|---|
| D-1 | Stateless: no state database, no outbox, no retries after exit; one `log.jsonl`. See [Files](interface.md#files), [Log](interface.md#log). | Deduplication, quiet hours and history belong to the routing tool (user decision S-2). |
| D-2 | Common input contract: `title`, `message`, `level`, `link`; unknown fields refused; text from stdin, a file or flags, all sanitized alike. See [Notification](interface.md#notification). | Architecture 13.2 rule 2; short argv text is untrusted (S-3). |
| D-3 | Named for what it is: `wsl-notice` (renamed from `desktop-notice`). | A tool tied to one platform's mechanism says so; siblings get their own names (user decision S-4). |
| D-4 | Notification text never touches argv or the environment: a constant `-EncodedCommand` script and one base64 JSON line on stdin. See [Toast construction](interface.md#toast-construction). | Architecture 13.2 rule 2; `powershell.exe`'s text would otherwise be visible in process lists on both sides (spike C checked `ps`, `/proc` and the Windows command line). |
| D-5 | The toast XML is built from text nodes (`InnerText`, `SetAttribute`), never by concatenation. | Naive concatenation let text inject images, looping audio and action buttons and broke on `<` and `&` (spike C). |
| D-6 | Base64 on stdin, not raw UTF-8. | The console input code page corrupted raw UTF-8 (spike C). |
| D-7 | `-EncodedCommand`, not `-Command -`. | `-Command -` runs stdin line by line, stops silently after the first statement of a multi-line block and gives no error exit (spike C). |
| D-8 | `link` becomes the toast's protocol-activation target, `http` and `https` only. | The toast API accepts any scheme (`file:`, `ms-settings:` and custom protocols built without validation). |
| D-9 | Output says `sent`, never `displayed`; exit 12 `toast_failed` only when PowerShell fails. | Display, dismissal and Focus Assist suppression are not observable from PowerShell 5.1 (spike C). |
| D-10 | One immediate retry on the interop error `UtilAcceptVsock`; a timeout is `outcome_unknown` with no retry. | The error appeared once after a hung or killed call and the next call worked; a retry after a timeout could show two toasts. |
| D-11 | Limits sit below what Windows accepted (title 100, body 1000 by default). | 5000-character bodies were accepted but on-screen truncation is not observable. |
| D-12 | Default `app_id` is the Windows PowerShell identity; unregistered identities are allowed by config but warned about in the docs. | The PowerShell identity needs no registration or install; unregistered ones produced no error and possibly no toast. |
| D-13 | Exit codes and outcomes as in [Exit codes](interface.md#exit-codes); exit 22 for a disabled tool; refusal `not_wsl` outside WSL. | As the sibling notifiers (S-7). |
| D-14 | No hooks in 0.x and no audit file; `config_invalid` is exit 2. See [Hooks](interface.md#hooks), [Log](interface.md#log). | As in slack-notice D-9 and D-6. |

## Resolved open topics

- Payload shape: [Notification](interface.md#notification).
- Channel config: [Config](interface.md#config); the tool reads no credentials.
- Escaping: handled by construction on the Windows side, plus control and invisible character removal ([Sanitizer](interface.md#sanitizer)).

## Live verification

Verified in design (private spike, 2026-11-17, Windows PowerShell 5.1 under WSL2; the tool's own constant script and fixtures):

| Behavior | Result |
|---|---|
| Interop | `powershell.exe` reachable; about 0.8 to 0.95 s per start with `-NoProfile`; no `pwsh.exe`, BurntToast or `wsl-notify-send.exe` present. |
| Toast API | WinRT toast works without a module under the PowerShell identity; exit 0. |
| Text transport | Base64 UTF-8 JSON on stdin; marker text absent from WSL `ps`, `/proc/<pid>/environ` and the Windows command line. |
| Escaping | `InnerText` renders `<`, `&`, `]]>`, quotes and a fake `</text><image …>` literally; NUL and C0 controls make it fail. |
| Limits | 5000-character body, 600-character title and empty body all exit 0. |
| Observability | Display and dismissal not observable. |
| Timeout | Terminating the child left no Windows PowerShell process. |

Still to verify (live tests and a human look, conservative defaults until then): that a toast under the default identity actually appears on screen (the user saw none from the spike run, whose identities included an unregistered one); how Windows truncates long titles and bodies; Focus Assist behavior; whether the `UtilAcceptVsock` retry is enough; behavior with PowerShell on a non-default path; WSL1.

## Modules

| Module | Responsibility |
|---|---|
| `cli.py` | Argument parsing, command dispatch, exit-code mapping, output. |
| `config.py` | Config loading, drop-ins, env overrides, Pydantic validation. |
| `payload.py` | Input parsing, notification validation, sanitizer. |
| `render.py` | Title and body assembly, truncation, the JSON line for stdin. |
| `toast.py` | The constant PowerShell script, process start, result mapping, the single retry. |
| `platform.py` | WSL detection and PowerShell resolution. |
| `log.py` | Log record writing and rotation. |
| `gateway.py` | The only place that starts processes and reads clocks; tests substitute a fake `powershell.exe`. |

## Data model, concurrency, errors

No stored objects and no locks. Errors use the envelope in [Error envelope](interface.md#error-envelope); log failures never change an exit code. The tool signals only the exact child it started and the process group it created.

## Test strategy

- All process starts go through `gateway.py`; a fake `powershell.exe` in `tests/fakebin/` records its argv, stdin and environment and returns each row of the result table in [Toast construction](interface.md#toast-construction).
- A golden test pins the constant script and asserts the argument list is identical for any notification; tests assert that no notification text appears in argv or the environment, with the spike's marker strings (quotes, `$`, backticks, newlines, emoji, `<`, `&`).
- Golden files for `--json` output and the stdin line, schema tests with positive and negative samples, a sanitizer table test.
- `live` tests (never in CI, WSL only) show marked toasts and run the injection fixtures through real PowerShell; they need a person to confirm display.

## Platforms

WSL2 on Windows 10 or 11 only; the tool refuses elsewhere.
