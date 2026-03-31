# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**agent-shell** is a bash-based sandbox for running coding agents (e.g., Claude Code). It uses [bubblewrap](https://github.com/containers/bubblewrap) (`bwrap`) to create an isolated filesystem/network namespace, with outbound network traffic filtered through a dedicated Squid HTTP proxy that enforces a domain whitelist.

## Installation

There is no build step. Install by running:

```bash
./install
```

This copies scripts to `~/bin`, config files to `~/etc`, and the systemd unit to `~/.config/systemd/user/`. It also creates required directories under `~/var/`.

After installing, enable and start the proxy service:

```bash
systemctl --user enable agent-shell-squid.service
systemctl --user start agent-shell-squid.service
```

## Architecture

The sandbox is a two-process system:

**Host side — `agent-shell-squid-proxy`** (run as a systemd user service):
- Starts a `socat` relay listening on a Unix socket (`~/var/run/agent-shell-proxy.sock`) and forwarding to Squid on TCP port 10385
- Starts `squid` (foreground) using `~/etc/agent-shell-squid.conf`, which restricts destinations to `~/etc/allowed_domains.txt`

**Inside the sandbox — `agent-shell`** (the main script):
- Uses `bwrap` with `--unshare-net` (and other namespace flags) so the sandbox has no real network
- Mounts the host Unix socket into the sandbox at `/run/user/$(id -u)/agent-shell-proxy.sock`
- Sets `http_proxy`/`https_proxy` to `http://127.0.0.1:8080`
- On entry, starts a `socat` listener on port 8080 inside the sandbox that forwards to the Unix socket, bridging the gap to the host Squid instance
- Binds the current working directory to `/workspace` inside the sandbox
- Binds `.agent-shell/` (created in the CWD) as `/home/sandbox` inside the sandbox

**Network flow:** Agent → port 8080 (socat inside sandbox) → Unix socket → socat on host → Squid (port 10385) → internet (whitelisted domains only)

## Key Files

| File | Purpose |
|------|---------|
| `src/agent-shell` | Main entry point; parses options, builds `bwrap` args, launches sandbox |
| `src/agent-shell-squid-proxy` | Host-side proxy starter (socat + squid) |
| `conf/agent-shell-squid.conf` | Squid config template (`HOME` is replaced by `./install`) |
| `conf/allowed_domains.txt` | Domain whitelist for Squid (Anthropic, GitHub, crates.io, rustup, docs.rs) |
| `conf/agent-shell-squid.service` | systemd user unit for the proxy |
| `install` | Installation script |

## Usage

```bash
agent-shell [OPTIONS] [-- COMMAND...]
```

Key options:
- `-A KEY` / `--api-key=KEY` — pass Anthropic API key into sandbox
- `-C` / `--mount-claude-md` — mount `~/.claude/CLAUDE.md` into sandbox
- `-r` / `--enable-rust` — mount `~/.cargo` and `~/.rustup` into sandbox
- `-X host:sandbox` / `--extra-mount=host:sandbox` — add an extra read-only bind mount

## Dependencies

- `bubblewrap` (`bwrap`)
- `socat`
- `squid`
