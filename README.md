# agent-safe

A lightweight bubblewrap sandbox for AI coding agents ([Pi](https://github.com/carderne/pi), [Pi Web](https://github.com/agegr/pi-web), [Antigravity](https://antigravity.google), [Claude Code](https://claude.ai/code), [Codex](https://github.com/openai/codex), [GitHub Copilot](https://github.com/github/copilot-cli), [Aider](https://aider.chat), and [OpenCode](https://opencode.ai)).

## Why

AI coding agents can write or delete files anywhere on your system, including outside the active project. Rather than relying on constant interactive permission prompts for every command, `agent-safe` isolates the filesystem with Bubblewrap so agents can safely run in full YOLO / skip-permissions mode without endangering the host system.

## What

`agent-safe` runs the agent in a Bubblewrap sandbox where:
- The host filesystem is mounted **read-only**.
- The active project directory is **writable**.
- Agent configuration and session state directories (`~/.pi`, `~/.gemini`, `~/.claude`, `~/.codex`, `~/.copilot`, `~/.aider`, `~/.config/opencode`) are **writable**.
- Package manager and compiler caches (Rust, Go, Node, Python, JVM, .NET, etc.) are **writable** so builds and dependency downloads persist without polluting outside directories.
- Host credentials (`~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.kube`, `~/.docker`) are **masked with tmpfs** to prevent unauthorized access (while keeping `$SSH_AUTH_SOCK`, `~/.ssh/known_hosts`, `~/.ssh/config`, and `~/.ssh/*.pub` public keys available for git operations and ssh-agent identity matching).
- Container engine sockets (`docker.sock`, `podman.sock`, `containerd.sock`) are **masked with `/dev/null`** and `DOCKER_HOST`/`CONTAINER_HOST` are unset. Because connecting to a UNIX domain socket ignores read-only filesystem mounts, an unmasked socket allows an agent to command the host daemon to spawn containers mounting the host filesystem read-write, completely bypassing sandbox write protections and secret masking.
- `/tmp` and `/dev/shm` are mounted as **tmpfs** (enabling shared memory for Playwright, Chromium, and test runners).
- The sandbox gets its own **PID namespace** (`--unshare-pid`), so an agent running `pkill`/`killall` cannot reach host processes, and it is torn down with the launching shell (`--die-with-parent`) so build daemons do not outlive the session.
- `~/.claude/ide` is **read-only**. Claude Code's editor extension publishes its WebSocket port, auth token and its own pid to `~/.claude/ide/<port>.lock`, and the CLI deletes any lockfile whose pid is not alive. That pid belongs to a host process, invisible inside the sandbox's PID namespace, so a sandboxed agent would declare every running editor dead and wipe the lockfiles — breaking IDE integration for host sessions too until the editor window is reloaded. Read-only makes that deletion fail harmlessly, and it also stops an agent planting a lockfile that would aim a host session's IDE connection at a server it controls. Keeping the lockfile is not enough on its own: inside an editor terminal the CLI accepts a lockfile whose port matches `CLAUDE_CODE_SSE_PORT` and otherwise insists the recorded pid be a live ancestor of itself, which no host pid can be in the sandbox. The extension exports that variable into its terminals but picks a fresh port on every activation, so in a terminal opened before the last window reload it names a dead port. `agent-safe` therefore resolves the lockfile for the project itself — on the host, where pids are visible — and pins `CLAUDE_CODE_SSE_PORT` to that port, keeping any live value the extension already exported. One limitation remains: reloading the editor window mid-session moves the port, and a sandboxed session cannot pick up the new one until it is restarted.
- Pi's IDE extensions have the same problem, and there are two of them: `npm:pi-x-ide` (`~/.pi/pi-x-ide/lock/<ide>-<pid>-<port>.lock`) and `npm:pi-ide-integration` (`~/.pi/ide/<port>.lock`). Both ignore a lockfile whose recorded pid is not alive, so both go blind in the sandbox; `pi-x-ide` also deletes what it rejects, and `~/.pi` is writable, so it wiped the host's lockfiles too. Neither has a usable port variable — `pi-ide-integration`'s `PI_IDE_PORT` skips the liveness test but then hands over a connection with no auth token, because the token still comes from the rejected file. What both do accept is a lockfile with no pid at all, so `agent-safe` binds a sanitized snapshot over each lock directory: the lockfiles whose pid is alive on the host, copied with the pid dropped. The originals stay out of the sandbox's reach, stale lockfiles are left out rather than sanitized, and the snapshots live in `$XDG_RUNTIME_DIR` for the lifetime of the session. The two are independent — separate directories, separate snapshots — so having both installed is fine. They are snapshots, so the same restart caveat applies.
- Git `hooks/` **and** `.git/config` are enforced as **read-only** in every repo layout (plain repos, worktrees, subdirectories, submodule gitdirs), so a sandboxed agent cannot leave behind anything that later runs on the host. Read-only hooks alone are not enough: `core.hooksPath`, `core.sshCommand`, `core.pager`, `alias.*`, `filter.*.clean/smudge` and `diff.*.textconv` all name commands git executes on the host. Per-worktree `config.worktree` files are covered too, since `core.hooksPath` is honored there whenever `extensions.worktreeConfig` is on. The tradeoff is that any command writing repo config must be run outside the sandbox: `git config`, `git remote add`, and also `git submodule update --init` (it records `submodule.<name>.url`), `git lfs install` and `git maintenance start`.

## Prerequisites

```bash
sudo pacman -S bubblewrap ripgrep jq
```

> Example is for CachyOS/Arch-based systems. For other distros, install `bubblewrap`, `ripgrep`, and `jq` via your package manager.

## Install

Download `agent-safe` into `~/.local/bin/` (or any directory in your `PATH`):

```bash
curl -o ~/.local/bin/agent-safe https://raw.githubusercontent.com/philong/agent-safe/main/agent-safe
chmod +x ~/.local/bin/agent-safe
```

### Symlink Shortcuts

Create symlinks to invoke each agent directly:

```bash
ln -sf ~/.local/bin/agent-safe ~/.local/bin/pi-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/pi-web-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/agy-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/claude-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/codex-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/copilot-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/aider-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/opencode-safe
```

## Usage

### 1. Pi CLI (`pi`)
Runs `pi` in yolo mode inside the sandbox:
```bash
cd ~/coding-projects/<project>
pi-safe
# or: agent-safe pi
```

### 2. Pi Web (`pi-web`)
Runs `pnpx @agegr/pi-web@latest --hostname 0.0.0.0` in yolo mode:
```bash
pi-web-safe [optional-project-dir]
# or: agent-safe pi-web [optional-project-dir]
```

### 3. Antigravity CLI (`agy`)
Runs `agy --dangerously-skip-permissions` inside the sandbox (automatically disables `enableTerminalSandbox` during the session to avoid nested namespace conflicts, and restores your setting on exit):
```bash
agy-safe
# or: agent-safe agy
```

### 4. Claude Code CLI (`claude`)
Runs `claude --dangerously-skip-permissions` inside the sandbox:
```bash
claude-safe
# or: agent-safe claude
```

### 5. Codex CLI (`codex`)
Runs `codex --dangerously-bypass-approvals-and-sandbox` inside the sandbox:
```bash
codex-safe
# or: agent-safe codex
```

### 6. GitHub Copilot CLI (`copilot`)
Runs `copilot --yolo` inside the sandbox:
```bash
copilot-safe
# or: agent-safe copilot
```

### 7. Aider (`aider`)
Runs `aider --yes` inside the sandbox:
```bash
aider-safe
# or: agent-safe aider
```

### 8. OpenCode CLI (`opencode`)
Runs `opencode --auto` inside the sandbox:
```bash
opencode-safe
# or: agent-safe opencode
```

### 9. Interactive Sandboxed Shell
Spawn an interactive shell inside the exact sandbox environment:
```bash
agent-safe shell
# or: agent-safe bash
```

### 10. Arbitrary Commands
Run any tool or command inside the sandbox:
```bash
agent-safe run cargo test
# or: agent-safe run python build.py
```

## Flags & Options

Options must come **before** the target. Everything after the target is passed
through to it untouched, so `agent-safe claude --help` shows Claude's help and
`agent-safe run rg --dry-run` passes `--dry-run` to `rg`.

| Flag | Description |
|---|---|
| `-C, --cwd <dir>` | Set project working directory without `cd`-ing first |
| `-w, --write <dir>` | Mount an extra directory as writable (e.g. sibling repo or dependency) |
| `--offline`, `--no-net` | Run with network isolation (`--unshare-net`) |
| `--dry-run` | Print sandbox mounts and `bwrap` command without executing |
| `--no-mask` | Disable secret masking (allows reading `~/.aws`, `~/.kube`, etc.) |

### Extra Writable Paths (Environment Variable)

You can also use an environment variable for extra write mounts:

```bash
AGENT_SAFE_WRITE=/path/one:/path/two agent-safe <target>
```

## Notes

- Network access is enabled by default (use `--offline` to disable).
- If running inside an existing container or sandbox where nested Bubblewrap is unavailable, `agent-safe` automatically detects it and falls back to running the agent directly.
- Do not install the `pi-sandbox` extension when using `agent-safe`, as it conflicts.

## Disclaimer

Use at your own risk. Bubblewrap is a low-level tool and only as secure as its configuration.
