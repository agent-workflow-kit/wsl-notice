# wsl-notice

Shows a Windows toast notification from a Linux process running inside WSL.

## Status

Designed (interface 0.1.0-design); not yet implemented. The normative specification is [docs/interface.md](docs/interface.md); decisions are in [docs/design.md](docs/design.md).

## Layer

L1 (primitive). Works fully standalone; it does not need the `awkit` harness.

## What it does

- Shows one Windows toast per call, through Windows PowerShell reached by WSL interop ([Toast construction](docs/interface.md#toast-construction)).
- Takes a JSON [notification](docs/interface.md#notification) with `title`, `message`, `level` (`info`, `warn`, `action`, `error`) and `link`, from stdin or a file, or the same fields as flags for human use.
- Sanitizes the text and cuts it to the configured limits ([Sanitizer](docs/interface.md#sanitizer), [Rendering](docs/interface.md#rendering)). The text never becomes markup: the toast is built from XML text nodes on the Windows side, so it cannot add images, sounds or buttons.
- `link`, when given (printable ASCII, `http` or `https`), is the toast's tap target; whether a tap opens it has not been tried yet. An optional `link_hosts` config key restricts the hosts.
- Keeps no database, only a `log.jsonl` that never holds a title, message or link. It runs no hooks and decides nothing about when or how often to notify; that belongs to its caller.

wsl-notice is the WSL member of a planned family of platform notifiers. Siblings for native Windows, other Linux desktops and macOS are planned as separate tools named `win-notice`, `linux-notice` and `mac-notice`; there are no dates.

## Quick start

```sh
echo '{"title": "Build", "message": "Finished", "level": "info"}' | wsl-notice send --stdin-json
wsl-notice send --title "Build" --message "Failed" --level error --link https://example.com/run/1
wsl-notice send --file note.json --dry-run --json
wsl-notice validate
wsl-notice validate --probe --test-message   # shows one test toast; look at your screen
```

## Commands

| Command | Purpose |
|---|---|
| `wsl-notice send [--stdin-json \| --file <path>\|-] [--title <t>] [--message <m>] [--level <l>] [--link <url>] [--dedupe-key <k>]` | Show one toast. |
| `wsl-notice validate [--probe] [--test-message]` | Check the config, WSL and PowerShell; `validate` never prompts. `--probe` also starts PowerShell once to print its version, `--test-message` shows one test toast (look at your screen; it needs no terminal). Exits 0 when every check passes, otherwise with the exit code of the failing check. |

Global flags: `--json`, `--config`, `--dry-run`, `--actor`, `--decision-id`, `--verbose`, `--quiet`, `--version` ([Global flags](docs/interface.md#global-flags)).

## How text is passed

Notification text never appears in a command line or in an environment variable. wsl-notice starts PowerShell with a constant argument list and sends the notification on stdin as one base64 line, so the text is not visible in process lists on either side. Flag text (`--title`, `--message`) is for humans typing at a terminal; callers pass JSON on stdin or in a file.

## Exit codes

0 handed to Windows (or a dry run); 1 internal error; 2 usage or invalid config; 12 failed (`not_wsl`, PowerShell missing, toast failed, interop failed, timeout with unknown outcome); 20 the payload was refused, code `refused` with a `reason` (`invalid_payload`, `empty_message`, `link_invalid`, `link_scheme`); 22 disabled in config (a successful no-op, `{"outcome": "disabled"}`). Every `send` that reaches input validation and ends in `sent`, a refusal or a failure is logged; dry runs and exit 22 are not. See [Exit codes](docs/interface.md#exit-codes).

## Limits

- The call completes with exit 0 and the toast is added to the Windows notification history, but on-screen display under the default identity is not yet confirmed; 0.1.0 is gated on a person seeing one. Output says `sent`, never `displayed`. Windows does not tell PowerShell whether a toast appeared, was dismissed or was suppressed by Focus Assist or do-not-disturb, so a person has to look. An unregistered `app_id` can show nothing without an error.
- Each call starts a Windows process, about one second.
- With `link_hosts` unset a producer can aim the tap target at any web address; the toast does not show the URL.
- Other processes of the same Windows user can read the notification from the notification history; do not send secrets.
- Not supported: native Windows, Linux desktops and macOS (separate tools), PowerShell 7, custom icons, sounds, buttons, images and progress bars. See [Residual risks](docs/interface.md#residual-risks).

## Config, hooks and environment

- Config: `~/.config/wsl-notice/config.yaml` plus `config.d/*.yaml`; every key is documented in [config.example.yaml](config.example.yaml) and [Config](docs/interface.md#config).
- Hooks: none in 0.x ([Hooks](docs/interface.md#hooks)).
- Environment: there are no `WSL_NOTICE_<KEY>` overrides; precedence is flags, config, defaults. wsl-notice reads no credentials ([Environment variables](docs/interface.md#environment-variables)).
- Calling from a script or agent: see [Calling from automation](docs/interface.md#calling-from-automation).

## Requirements

- WSL2 on Windows 10 or 11, with Windows PowerShell 5.1 reachable through WSL interop (`[interop] enabled=true` in `/etc/wsl.conf`). `powershell_bin` is an absolute path with no `PATH` search; the default `/mnt/c/Windows/System32/WindowsPowerShell/v1.0/powershell.exe` assumes the default automount root, so set it in config if yours differs. Outside WSL the tool exits 12 `not_wsl` (a dry run still works).
- Python 3.12 or newer and [uv](https://docs.astral.sh/uv/) to install it.

## Troubleshooting: no toast appears

`sent` only means Windows accepted the toast. If nothing shows on screen:

1. Run `wsl-notice validate --probe --test-message` and look at your screen.
2. Check Windows Settings, System, Notifications, and that "Windows PowerShell" is allowed to notify.
3. Check Focus Assist or Do Not Disturb.
4. Try another `app_id` in config; an identity that is not registered shows nothing and gives no error.
5. If `validate` reports a stale interop socket (a long-lived tmux or cron job), start the command from a fresh shell.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/wsl-notice
```

## Documentation

- [docs/interface.md](docs/interface.md): normative interface
- [docs/design.md](docs/design.md): decisions and internals
- [docs/plan.md](docs/plan.md): staged implementation plan
- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))

## License

MIT — see LICENSE.
