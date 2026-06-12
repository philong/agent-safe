# pi-safe

A simple bubblewrap sandbox for the [Pi](https://github.com/carderne/pi) AI coding agent.

## Why

Pi is a powerful coding agent, but it can write or delete files anywhere on your system, including outside the project you're working on. I didn't want to run Pi without a sandbox.

I tried [pi-sandbox](https://github.com/carderne/pi-sandbox) first, but it broke bash commands, which is essential for my workflows. So I built my own.

## What

pi-safe is a bash script that wraps Pi in bubblewrap. The entire filesystem runs read-only, except the current project directory and `~/.pi`. That covers both bash commands and Pi's built-in file tools, so the agent can't write or delete anything outside the active project.

`~/.pi` stays writable so you can build your own extensions, edit `AGENTS.md`, and enhance Pi.

## Prerequisites

```bash
sudo pacman -S bubblewrap ripgrep
```

> Example is for CachyOS/Arch-based. For other distros, install bubblewrap with your package manager.

## Install

The script lives in `~/.local/bin/`, the standard user scripts directory on Linux. Make sure it's in your `PATH`.

```bash
curl -o ~/.local/bin/pi-safe https://raw.githubusercontent.com/dimaginar/pi-safe/main/pi-safe
chmod +x ~/.local/bin/pi-safe
```

## Usage

```bash
cd ~/coding-projects/<project>
pi-safe
```

## Notes

- Full filesystem read access, no protection against data exfiltration
- Network unrestricted
- `~/.pi` is writable, required to build extensions, edit `AGENTS.md`, and enhance Pi
- Do not install the pi-sandbox extension, it conflicts with pi-safe

## Disclaimer

Use at your own risk. Bubblewrap is a low-level tool and only as secure as how you configure it. It does not protect against running untrusted code and network access remains unrestricted. Do your own research before using it in sensitive environments.
