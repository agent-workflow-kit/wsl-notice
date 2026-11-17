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
- `link`, when given and `http` or `https`, opens when the toast is clicked.
- Keeps no database, only a `log.jsonl` that never holds a title, message or link. It runs no hooks and decides nothing about when or how often to notify; that belongs to its caller.

wsl-notice is the WSL member of a planned family of platform notifiers. Siblings for native Windows, other Linux desktops and macOS are planned as separate tools named `win-notice`, `linux-notice` and `mac-notice`; there are no dates.

## Quick start

```sh
echo '{"title": "Build", "message": "Finished", "level": "info"}' | wsl-notice send --stdin-json
wsl-notice send --title "Build" --message "Failed" --level error --link https://example.com/run/1
wsl-notice send --file note.json --dry-run --json
wsl-notice validate
wsl-notice validate --probe --test-message
```

## Commands

| Command | Purpose |
|---|---|
| `wsl-notice send [--stdin-json \| --file <path>\|-] [--title <t>] [--message <m>] [--level <l>] [--link <url>] [--dedupe-key <k>]` | Show one toast. |
| `wsl-notice validate [--probe] [--test-message]` | Check the config, WSL and PowerShell; `--probe` also starts PowerShell once, `--test-message` shows a marked test toast. |

Global flags: `--json`, `--config`, `--dry-run`, `--yes`, `--actor`, `--verbose`, `--quiet`, `--version` ([Global flags](docs/interface.md#global-flags)).

## How text is passed

Notification text never appears in a command line or in an environment variable. wsl-notice starts PowerShell with a constant argument list and sends the notification on stdin as one base64 line, so the text is not visible in process lists on either side. Flag text (`--title`, `--message`) is for humans typing at a terminal; callers pass JSON on stdin or in a file.

## Exit codes

0 handed to Windows (or a dry run); 1 internal error; 2 usage or invalid config; 12 failed (PowerShell missing, toast failed, interop failed, timeout with unknown outcome); 20 refused before sending with a `reason` (invalid payload, empty message, bad link, not WSL); 22 disabled in config (a successful no-op). See [Exit codes](docs/interface.md#exit-codes).

## Limits

- Output says `sent`, never `displayed`. Windows does not tell PowerShell whether a toast appeared, was dismissed or was suppressed by Focus Assist or do-not-disturb, so a person has to look. An unregistered `app_id` can show nothing without an error.
- Each call starts a Windows process, about one second.
- Other processes of the same Windows user can read the notification from the notification history; do not send secrets.
- Not supported: native Windows, Linux desktops and macOS (separate tools), PowerShell 7, custom icons, sounds, buttons, images and progress bars. See [Residual risks](docs/interface.md#residual-risks).

## Config, hooks and environment

- Config: `~/.config/wsl-notice/config.yaml` plus `config.d/*.yaml`; every key is documented in [config.example.yaml](config.example.yaml) and [Config](docs/interface.md#config).
- Hooks: none in 0.x ([Hooks](docs/interface.md#hooks)).
- Environment: `WSL_NOTICE_<KEY>` for scalar config keys; wsl-notice reads no credentials ([Environment variables](docs/interface.md#environment-variables)).

## Requirements

- WSL2 on Windows 10 or 11, with Windows PowerShell 5.1 (`powershell.exe`) reachable through WSL interop. The tool refuses to run elsewhere (exit 20 `not_wsl`).
- Python 3.12 or newer and [uv](https://docs.astral.sh/uv/) to install it.

## Install

```sh
uv tool install git+https://github.com/agent-workflow-kit/wsl-notice
```

## Documentation

- [docs/interface.md](docs/interface.md): normative interface
- [docs/design.md](docs/design.md): decisions and internals
- [awkit architecture](../awkit-docs/architecture.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/architecture.md))
- [awkit design guide](../awkit-docs/design-guide.md) ([published](https://github.com/agent-workflow-kit/awkit-docs/blob/main/design-guide.md))

## License

MIT — see LICENSE.
