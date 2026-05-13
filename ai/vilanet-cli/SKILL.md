---
name: vilanet-cli
description: Drive the vilanet-cli VPN client end-to-end on Linux, macOS, and Windows — login, list packages/servers, connect, switch nodes, disconnect, inspect status, tail logs, change settings, and install/uninstall as a background service. Use whenever the user mentions VilaNet, vilanet-cli, the VilaNet VPN, "connect to my VPN", "switch nodes", "TUN privileges missing", a vilanet-cli exit code (1, 2, 3, 4, 5, 9), a settings key (e.g. routing_mode, tun.stack, hy2.speed_mode), or wants to set up VilaNet as a background service (systemd, LaunchAgent, Windows Service).
---

# vilanet-cli — operator skill for AI agents

Homepage: <https://github.com/vilavpn/vilanet-cli>

`vilanet-cli` is the cross-platform command-line VilaNet VPN client. One static
Go binary, no GUI, no system tray. This skill teaches you (the AI) how to
drive it for an end user.

## When to use this skill

- The user asks anything about VilaNet, vilanet-cli, or "my VilaNet VPN".
- The user wants to connect, disconnect, switch nodes, or check VPN status
  on their own machine and `vilanet-cli` is installed (or they want to
  install it).
- The user reports a vilanet-cli error or exit code.
- The user wants to change a persistent setting (TUN stack, routing mode,
  Hy2 speed, etc.) or set up a background service.

## When NOT to use this skill

- Generic VPN questions unrelated to VilaNet (OpenVPN, WireGuard, etc.).
- Server-side VilaNet operations (panel admin, node provisioning, billing).
  Those are out of scope for this CLI.
- VilaNet on OpenWRT, Asuswrt-Merlin, Apple TV, iOS, Android, or the Flutter
  desktop app — those are *sibling* clients with their own surfaces.

## Mental model

```
$ vilanet-cli login        # stores creds in OS keyring (or plaintext with --insecure-store)
$ vilanet-cli connect      # foreground; Ctrl-C disconnects
                           # OR --background to detach (Linux/macOS only — see Windows note)
$ vilanet-cli status       # asks the running daemon over a Unix socket (Linux/macOS only in v0.1.0)
$ vilanet-cli switch       # tells the daemon to restart on another node (Linux/macOS only in v0.1.0)
$ vilanet-cli disconnect   # idempotent
$ vilanet-cli logout       # clears keyring + caches
```

There is at most **one** daemon per user. On Linux and macOS, subcommands
talk to it over a Unix domain socket (`$XDG_CACHE_HOME/vilanet/cli.sock`
on Linux, `~/Library/Caches/com.viraltech.vilanet/cli.sock` on macOS).

**Windows v0.1.0 has no IPC control plane.** The named-pipe transport is
stubbed (`internal/ipc/client_windows.go` returns an unsupported error),
so `status`, `switch`, and `disconnect` cannot reach a service-managed
daemon — `status` always reports `running:false` even when the Windows
Service is up. To manage the Service on Windows, use Windows' Service
Control Manager (`sc.exe query VilaNetCLI`, `Stop-Service VilaNetCLI`,
`Restart-Service VilaNetCLI`) and `vilanet-cli logs` for visibility.
Named-pipe IPC is on the v0.2.0 roadmap.

If `vilanet-cli` is missing, the user must install it first
(see *Install* below).

## Quick start (most common task)

```bash
vilanet-cli login --email you@example.com           # password prompt
vilanet-cli list servers --country HK               # see what's available
vilanet-cli connect --country HK                    # foreground; Ctrl-C to stop
```

For a non-blocking proxy on Unix:

```bash
vilanet-cli connect --background --no-tun --mixed 127.0.0.1:1080
# Point browser/system proxy at SOCKS5 127.0.0.1:1080
```

## Global flags (apply to every subcommand)

| Flag              | Meaning                                                                |
|-------------------|------------------------------------------------------------------------|
| `--json`          | Machine-readable JSON output (where supported)                         |
| `--debug`         | Verbose debug logging to stderr                                        |
| `--insecure-store`| Plaintext credential file instead of OS keyring (prints a warning)     |
| `--config DIR`    | Override config directory (defaults to OS-appropriate user config dir) |

Always prefer `--json` when parsing output to feed into the next step.

## Command reference

### `login`

```
vilanet-cli login [--email <addr>] [--password <pw>]
```

Stores credentials in the OS keyring (Keychain / Secret Service / Windows
Credential Manager). Omit `--password` to be prompted (preferred). Never
pass `--password` literally on the command line in an AI session — quote it
to the user and let them type it, or instruct them to omit it.

### `list packages` / `list servers`

```
vilanet-cli list packages [--json]
vilanet-cli list servers [--package <id>] [--country <ISO2>] [--json]
```

`--country` is case-insensitive (`hk`, `HK`, `Hk` all work). `--package` is
the package id from `list packages`.

### `connect`

```
vilanet-cli connect [--auto | --country <ISO2> | --node <hex16>]
                    [--background]
                    [--no-tun] [--mixed HOST:PORT]
                    [--clash-port N] [--clash-secret <str>]
```

- `--auto` overrides `--node` and `--country` and picks globally.
- `--node` takes the 16-char hex node id from `list servers --json`.
- `--background` detaches into a daemon. Not supported on Windows v0.1.0 —
  exits **9** with a hint to install the Windows Service.
- `--no-tun` skips the TUN inbound; combine with `--mixed 127.0.0.1:PORT`
  to expose a SOCKS5+HTTP proxy on loopback. Useful when (a) you can't
  elevate and (b) you only need a per-app proxy.

### `switch`

```
vilanet-cli switch --node <hex16>
```

Tells the running daemon to restart on a different node. **Brief
packet-loss window (~1–2 s)** while sing-box restarts. Preserves the
credential cache, so it's faster than `disconnect && connect`.

### `disconnect`

```
vilanet-cli disconnect
```

Idempotent — exits 0 even if no daemon is running. Always safe.

### `status`

```
vilanet-cli status [--json] [--stats]
```

Reads daemon state over IPC. With `--stats`, includes Clash API traffic
counters (best-effort; absence is not an error). With no daemon, prints a
stopped-state stub and exits 0.

### `settings`

```
vilanet-cli settings list                    # every key + current value
vilanet-cli settings get [KEY]               # one key, or all if KEY omitted
vilanet-cli settings set KEY VALUE           # persist to config.json
```

See *Settings keys* below.

### `logs`

```
vilanet-cli logs [--lines N] [--follow]
```

Prints the last N lines (default 50; `-1` for the whole file) and optionally
tails. Poll-based at 200 ms. Log paths per platform:

- Linux: `${XDG_STATE_HOME:-~/.local/state}/vilanet/logs/`
- macOS: `~/Library/Logs/com.viraltech.vilanet/`
- Windows: `%LOCALAPPDATA%\VilaNet\Logs\`

### `logout`

```
vilanet-cli logout [--purge]
```

Clears keyring credentials and ephemeral caches. `--purge` also deletes
`config.json`, resetting settings to defaults on next run.

## Settings keys

Persisted to `${ConfigDir}/config.json`. Defaults shown.

| Key                       | Default                       | Notes                                                                  |
|---------------------------|-------------------------------|------------------------------------------------------------------------|
| `dns1`                    | `https://1.0.0.1/dns-query`   | Primary DNS — DoH URL or plain IP                                      |
| `dns2`                    | `223.5.5.5`                   | Secondary DNS                                                          |
| `domain_strategy`         | `prefer_ipv4`                 | One of `prefer_ipv4`, `prefer_ipv6`, `ipv4_only`, `ipv6_only`         |
| `routing_mode`            | `rule`                        | `rule` (China bypass) or `global` (route everything through tunnel)    |
| `block_ads`               | `false`                       | Apply ad-block ruleset                                                 |
| `block_porn`              | `false`                       | Apply adult-content ruleset                                            |
| `hy2.speed_mode`          | `off`                         | `off`, `server_default`, `custom` — Hysteria2 bandwidth hint           |
| `hy2.custom_mbps`         | `0`                           | Only when `hy2.speed_mode=custom`                                      |
| `hop_interval`            | `10`                          | Hysteria2 port-hop interval in seconds                                 |
| `tun.enabled`             | `true`                        | Bring up TUN inbound on `connect`                                      |
| `tun.ipv6`                | `false`                       | Dual-stack TUN                                                         |
| `tun.stack`               | `mixed`                       | `system`, `gvisor`, `mixed`                                            |
| `tun.mtu`                 | `9000`                        | TUN MTU                                                                |
| `mixed.enabled`           | `false`                       | Expose mixed SOCKS5+HTTP inbound persistently                          |
| `mixed.listen`            | `127.0.0.1`                   | Bind address for mixed inbound                                         |
| `mixed.port`              | `10081`                       | Bind port for mixed inbound                                            |
| `clash_api.enabled`       | `true`                        | Local Clash API for stats / external control                           |
| `clash_api.port`          | `9090`                        | Clash API port                                                         |
| `clash_api.secret`        | `""`                          | Clash API bearer secret (empty = no auth)                              |
| `selected_package`        | `""`                          | Sticky package id                                                      |
| `selected_server`         | `""`                          | Sticky node id                                                         |

If a `settings set` call you propose isn't in this table, re-run
`vilanet-cli settings list` first and update — the table is authoritative
for the **current** build only.

## Exit codes — diagnosis

Stable across releases. Shell scripts and AI tools may rely on these.

| Code | Meaning                                       | First fix to suggest                                                                                  |
|------|-----------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 0    | Success                                       | —                                                                                                     |
| 1    | Generic error (bad flag, IO, JSON encoding)   | Re-read the stderr line; if `--debug` is missing, retry with it.                                      |
| 2    | Authentication failed                         | `vilanet-cli login` again. Check email / password.                                                    |
| 3    | Account closed                                | Tell the user to contact VilaNet support — the CLI cannot fix this.                                   |
| 4    | Rate-limited **or** no stored credentials     | Run `vilanet-cli login`. If that succeeded recently, wait the rate-limit window (minutes).            |
| 5    | TUN privileges missing                        | Linux: `sudo setcap cap_net_admin,cap_net_bind_service+eip $(command -v vilanet-cli)`. macOS: run with `sudo`. Windows: launch elevated, or use Windows Service. Or fall back to `--no-tun --mixed 127.0.0.1:1080`. |
| 9    | `--background` not supported on Windows       | Install the Windows Service (`packaging/windows/install-service.ps1`).                                |

## Troubleshooting playbook

Diagnose top-down. **Always** call `vilanet-cli status --json` first when
something seems wrong — it's cheap and tells you whether a daemon is alive.

| Symptom                                                  | Diagnostic                                       | Fix                                                                                              |
|----------------------------------------------------------|--------------------------------------------------|--------------------------------------------------------------------------------------------------|
| `connect` exits 5                                        | —                                                | See exit-code 5 row above                                                                        |
| `connect` exits 4                                        | `vilanet-cli status --json`                      | Likely no creds → `login`. If recently logged in → rate-limit, wait.                            |
| `connect` hangs                                          | `vilanet-cli logs --lines 200`                   | Look for TLS / DNS errors; suggest swapping `dns1` to a reachable resolver                       |
| `switch` says "no daemon"                                | `vilanet-cli status`                             | Run `connect` first; `switch` only swaps a running daemon                                        |
| `disconnect` "no daemon" but tunnel still active         | `pgrep -af vilanet-cli`                          | Stale orphan: `kill <pid>` then `vilanet-cli disconnect` again                                   |
| `bind: address already in use` (mixed / Clash port)      | `vilanet-cli logs`                               | Change `mixed.port` / `clash_api.port` via `settings set`                                        |
| macOS LaunchAgent crash-loop                             | `log show --predicate 'process == "vilanet-cli"' --last 5m` | LaunchAgent runs as user — must use `--no-tun --mixed`, NOT TUN                            |
| Linux service starts but no IPC                          | `systemctl status vilanet-cli@$USER`             | Confirm capability set: `getcap $(command -v vilanet-cli)` should list `cap_net_admin`           |
| Windows: tunnel up but apps don't route                  | —                                                | Confirm running as Administrator; check Windows Service status via Services.msc                  |

## Install / uninstall

### Linux

Release archive names are versioned (`vilanet-cli_<version>_<os>_<arch>.tar.gz`),
so use the GitHub CLI with a wildcard pattern for a stable command line.
Replace the `--repo` value if the release lives in a different org/fork:

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_linux_amd64.tar.gz' --clobber
tar -xzf vilanet-cli_*_linux_amd64.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
sudo setcap 'cap_net_admin,cap_net_bind_service+eip' /usr/local/bin/vilanet-cli
# Optional per-user systemd template (requires a source checkout):
sudo cp packaging/systemd/vilanet-cli@.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vilanet-cli@$USER
```

Uninstall:

```bash
sudo systemctl disable --now vilanet-cli@$USER 2>/dev/null
sudo rm -f /etc/systemd/system/vilanet-cli@.service /usr/local/bin/vilanet-cli
```

### macOS

From a release tarball (binary only — uses `gh` to handle versioned asset names):

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_darwin_arm64.tar.gz' --clobber   # or _darwin_amd64 on Intel
tar -xzf vilanet-cli_*_darwin_*.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
```

From a source checkout (also installs the optional LaunchAgent that runs
`--no-tun --mixed 127.0.0.1:10081` at login — needs `bin/vilanet-cli` to
exist, so run `make build` first):

```bash
make build
sudo bash scripts/install-macos.sh --agent
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.viraltech.vilanet.cli.plist
launchctl kickstart  -k gui/$(id -u)/com.viraltech.vilanet.cli
```

Uninstall:

```bash
launchctl bootout gui/$(id -u)/com.viraltech.vilanet.cli 2>/dev/null
sudo rm -f /usr/local/bin/vilanet-cli ~/Library/LaunchAgents/com.viraltech.vilanet.cli.plist
```

The LaunchAgent uses `--no-tun --mixed` because a per-user agent cannot get
TUN privileges. For TUN at boot, install a `/Library/LaunchDaemons` plist
that runs the binary as root.

### Windows

Download the latest zip (versioned name — use `gh release download` if
you have GitHub CLI installed):

```powershell
gh release download --repo vilavpn/vilanet-cli `
    --pattern 'vilanet-cli_*_windows_amd64.zip' --clobber
Expand-Archive -Force vilanet-cli_*_windows_amd64.zip .
```

Then either:

1. Run from an **elevated** PowerShell:
   ```powershell
   vilanet-cli.exe connect
   ```
2. Or install as a Service (Administrator):
   ```powershell
   New-Item -ItemType Directory -Force "$Env:ProgramFiles\VilaNet" | Out-Null
   Copy-Item .\vilanet-cli.exe "$Env:ProgramFiles\VilaNet\vilanet-cli.exe"
   .\packaging\windows\install-service.ps1
   Start-Service VilaNetCLI
   ```

Uninstall:

```powershell
Stop-Service VilaNetCLI -ErrorAction SilentlyContinue
sc.exe delete VilaNetCLI
Remove-Item "$Env:ProgramFiles\VilaNet" -Recurse -Force
```

**Windows v0.1.0 does NOT support `--background`** (exit 9). Use the Service
instead.

## Working with `--json`

Output shapes are flat — verified against `internal/ipc/protocol.go` and
`vilanet-core/auth/public.go`. Don't assume nested objects.

`vilanet-cli status --json --stats` when a daemon is up:

```json
{
  "running": true,
  "pid": 12345,
  "node": "auto/HK",
  "connected": true,
  "start_time": "2026-05-13T09:21:33Z",
  "upload_bytes": 6789,
  "download_bytes": 12345
}
```

When no daemon is up, it prints `{"running":false}` and exits 0. `node`
is the daemon's *display selector* — a free-form string returned by
`displaySelection()` — so it can be `auto` (global auto-select),
`auto/<ISO2>` (country-scoped auto), or a 16-char hex node id when the
user pinned one. Treat it as opaque; don't pattern-match for a hex id.

`upload_bytes` and `download_bytes` are only populated when `--stats` is
passed (they overlay the Clash-API counters). Plain `status --json`
omits both fields.

`vilanet-cli list servers --json` returns an array of:

```json
{ "package": "<pkg-id>", "id": "<node-id-hex>", "name": "hk-01",
  "type": "hysteria2", "country": "HK" }
```

`vilanet-cli list packages --json` returns an array of:

```json
{ "id": "<pkg-id>", "name": "Asia Hy2",
  "due_date": "2026-12-31", "system_type": "hy2",
  "node_count": 12, "nodes": [ /* PublicNode[] as above, minus the wrapping package field */ ] }
```

When chaining commands, prefer parsing `--json` over scraping human
output.

## Safety rules for the AI

1. **Never** print, log, or transmit the user's password. Always have them
   type it at the `login` prompt.
2. **Never** run `rm -rf` to "clean up" vilanet-cli. Use `logout --purge`
   and the documented uninstall steps.
3. **Always** surface non-zero exit codes verbatim to the user; do not
   silently retry.
4. Before destructive settings changes (`routing_mode global`, disabling
   TUN), confirm intent in one sentence.
5. If the daemon is not running, `status` and `disconnect` are safe; every
   other subcommand has user-visible side effects — prefer `status` first.

## Build from source (only if the user asks)

```bash
make build                      # bin/vilanet-cli
make test                       # unit tests
make smoke                      # build + scripts/smoke-test.sh
```

Go version per `go.mod` (currently 1.26.2+).

## Reference

- Repo (public release + skill mirror): https://github.com/vilavpn/vilanet-cli
- PRD: `docs/vilanet-cli-prd.md` in the repo
- Packaging guide: `docs/packaging.md`
- Architecture: `docs/architecture.md`
