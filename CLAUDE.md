# CLAUDE.md: wsl-notice

wsl-notice is an awkit L1 primitive: it shows a Windows toast from a process running inside WSL. It must work fully standalone (no harness, bus, shared library or shared contracts package).

**Naming.** Tools have no shared prefix; see design guide §1 naming rules. Platform siblings are separate tools (`win-notice`, `linux-notice`, `mac-notice`).

**Status.** Design done (interface 0.1.0-design); not yet implemented.

## Read first

1. `../awkit-docs/architecture.md`: this tool's §5.1 entry and its §20 entry (constraints and open topics), plus the cross-cutting sections listed in `docs/design.md`.
2. `../awkit-docs/design-guide.md`: the conventions every awkit tool follows.
3. `docs/interface.md`: the single normative source for this tool's rules.
4. `docs/design.md`: decisions, modules and test strategy; `docs/plan.md`: the staged plan.

## Rules

- Open source: never add confidential information (company names, hostnames, ticket project keys, repo or org names, people's names, private skills). Organization specifics belong in the user's config.
- Never import sibling repos. Read their docs and schemas by path only; copying a sibling's schema files into `tests/` pinned to a version is allowed.
- No upward awareness: this tool never knows about, or behaves differently for, the tools that call it. Behavior may differ only by explicit arguments.
- Interface first: `docs/interface.md` and `schemas/` change before the implementation does.
- Semver. A breaking interface change is, before 1.0, a `feat` commit whose title names the break (never `!` before 1.0), and from 1.0 on a `!` commit; the title becomes the changelog line (details go in the PR body). Either way, update `awkit-docs/interface-registry.md`. Never hand-edit CHANGELOG.md or the version.
- Run `make verify` before every commit.
- All subprocess calls, input reads, the home directory and uid, clocks and the interop sockets go through `src/wsl_notice/gateway.py`, so tests can substitute fakes; process tests call `cli.main(argv, env, gateway)` in-process, and a fake `powershell.exe` lives in `tests/fakebin/` for the few real-subprocess tests.
- Never run destructive git commands (reset, checkout --, clean, restore, stash drop, force-push). Never kill processes by pattern; signal only the exact child this tool started and the process group it created.
- The 0.1.0 release is gated on a person seeing a toast under the default identity (`docs/plan.md`, stage 0).
- No environment overrides and no `PATH` search for PowerShell: `powershell_bin` is an absolute path from config. The config is read only from the home directory of the password database (`--config` selects another file); neither `HOME` nor `XDG_CONFIG_HOME` is read, and a failed or unusable home lookup is exit 2 `config_invalid` (key `home`), never a fallback.
- Untrusted text travels only in files or stdin, never argv or env.
- Minimize dependencies; justify each one in `docs/design.md`.

## Workflow

1. The design is done (interface 0.1.0-design). Implement test-first; `docs/interface.md` wins over any other document.
2. Interface changes update `docs/interface.md`, `schemas/` and `../awkit-docs/interface-registry.md` together.
3. The constant PowerShell script is pinned by a golden test; never build the toast XML by string concatenation.

## Repository workflow

- Work in a git worktree on a branch named `<type>/<short-slug>`, never in the main checkout (design guide section 17).
- PR titles are Conventional Commit headers; they become the squash commit, the version bump and the changelog line (architecture section 18.3). Do not use `!` before 1.0 (a breaking change before then is a `feat` whose title names the break).
- Never use `--no-verify`. Check PR titles, bodies and comments with `guard text` where the private-content guard is installed (architecture section 18.7).
- Never merge a PR without the maintainer's explicit instruction for that PR; merges go through `awkit-ci`'s `scripts/pr-merge` (architecture section 18.6).
- Never hand-edit `CHANGELOG.md` or the version in `pyproject.toml`; the release job owns them.
