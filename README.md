# agent-safe

A lightweight bubblewrap sandbox for AI coding agents ([Antigravity](https://antigravity.google), [Claude Code](https://claude.ai/code), [Pi](https://github.com/carderne/pi), [Pi Web](https://github.com/agegr/pi-web), and [Aider](https://aider.chat)).

## Why

AI coding agents can write or delete files anywhere on your system, including outside the active project. Rather than relying on constant interactive permission prompts for every command, `agent-safe` isolates the filesystem with Bubblewrap so agents can safely run in full YOLO / skip-permissions mode without endangering the host system.

## What

`agent-safe` runs the agent in a Bubblewrap sandbox where:
- The host filesystem is mounted **read-only**.
- The active project directory is **writable**.
- Agent configuration and session state directories (`~/.pi`, `~/.gemini`, `~/.claude`, `~/.aider`) are **writable**.
- Package manager and compiler caches (Rust, Go, Node, Python, JVM, .NET, etc.) are **writable** so builds and dependency downloads persist without polluting outside directories.
- Host credentials (`~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.kube`, `~/.docker`) are **masked with tmpfs** to prevent unauthorized access (while keeping `$SSH_AUTH_SOCK` available for git operations).
- `/tmp` and `/dev/shm` are mounted as **tmpfs** (enabling shared memory for Playwright, Chromium, and test runners).
- Git worktrees are safely mounted with `hooks/` and `config` enforced as **read-only** so sandboxed commands cannot plant malicious Git hooks.

## Prerequisites

```bash
sudo pacman -S bubblewrap ripgrep jq
```

> Example is for CachyOS/Arch-based systems. For other distros, install `bubblewrap`, `ripgrep`, and `jq` via your package manager.

## Install

Download `agent-safe` into `~/.local/bin/` (or any directory in your `PATH`):

```bash
curl -o ~/.local/bin/agent-safe https://raw.githubusercontent.com/dimaginar/pi-safe/main/agent-safe
chmod +x ~/.local/bin/agent-safe
```

### Symlink Shortcuts

Create symlinks to invoke each agent directly:

```bash
ln -sf ~/.local/bin/agent-safe ~/.local/bin/agy-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/claude-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/aider-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/pi-safe
ln -sf ~/.local/bin/agent-safe ~/.local/bin/pi-web-safe
```

## Usage

### 1. Antigravity CLI (`agy`)
Runs `agy --dangerously-skip-permissions` inside the sandbox:
```bash
agy-safe
# or: agent-safe agy
```

### 2. Claude Code CLI (`claude`)
Runs `claude --dangerously-skip-permissions` inside the sandbox:
```bash
claude-safe
# or: agent-safe claude
```

### 3. Aider (`aider`)
Runs `aider --yes` inside the sandbox:
```bash
aider-safe
# or: agent-safe aider
```

### 4. Pi CLI (`pi`)
Runs `pi` in yolo mode inside the sandbox:
```bash
cd ~/coding-projects/<project>
pi-safe
# or: agent-safe pi
```

### 5. Pi Web (`pi-web`)
Runs `pnpx @agegr/pi-web@latest --hostname 0.0.0.0` in yolo mode:
```bash
pi-web-safe [optional-project-dir]
# or: agent-safe pi-web [optional-project-dir]
```

### 6. Interactive Sandboxed Shell
Spawn an interactive shell inside the exact sandbox environment:
```bash
agent-safe shell
# or: agent-safe bash
```

### 7. Arbitrary Commands
Run any tool or command inside the sandbox:
```bash
agent-safe run cargo test
# or: agent-safe run python build.py
```

## Flags & Options

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
# (PI_SAFE_WRITE is also supported for backward compatibility)
```

## Notes

- Network access is enabled by default (use `--offline` to disable).
- Do not install the `pi-sandbox` extension when using `agent-safe`, as it conflicts.

## Disclaimer

Use at your own risk. Bubblewrap is a low-level tool and only as secure as its configuration.
