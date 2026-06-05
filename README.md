# vilanet-cli

Cross-platform command-line VilaNet VPN client. One static binary, no GUI,
no system tray. Runs on **Linux**, **macOS**, and **Windows** (amd64, arm64,
and armv7 on Linux).

This repository is the **public distribution surface** — release binaries
and the AI Agent Skill. Source code lives in a private repository.

---

## What it does

- Connects to VilaNet through any supported protocol (Hysteria2,
  VLESS+Reality, VMess, Shadowsocks, Trojan) with the same node fleet as
  the mobile and desktop GUI apps.
- Embeds [sing-box](https://github.com/SagerNet/sing-box) as a Go library
  — no separate sing-box install needed.
- Runs foreground or detaches to a background daemon (Linux/macOS); installs
  as a Windows Service.
- **Auto-updates** by default — checks for new releases on connect, verifies
  Ed25519-signed checksums, and applies updates automatically. The
  force-update floor prevents running versions with known issues.
- **Secrets concealment** — entry IPs, API hostnames, and mechanism
  vocabulary are scrubbed from logs and config dumps so diagnostic output is
  safe to share.
- Exposes a `diag` subcommand that writes a redacted support bundle (log
  tail, settings, node list) to a single `.tar.gz` file.

For the full interactive user guide — install walkthrough, simulator,
settings reference, and troubleshooting — visit
**[cli.vilavpn.com](https://cli.vilavpn.com)**.

---

## Install

### Linux

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_linux_amd64.tar.gz' --clobber
tar -xzf vilanet-cli_*_linux_amd64.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
sudo setcap 'cap_net_admin,cap_net_bind_service+eip' /usr/local/bin/vilanet-cli
```

arm64 and armv7 builds are also available — replace `linux_amd64` with
`linux_arm64` or `linux_armv7`.

To run as a persistent background service, install the per-user systemd
template unit:

```bash
sudo cp packaging/systemd/vilanet-cli@.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now vilanet-cli@$USER
```

### macOS

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_darwin_arm64.tar.gz' --clobber
# On Intel Mac, use: --pattern 'vilanet-cli_*_darwin_amd64.tar.gz'
tar -xzf vilanet-cli_*_darwin_*.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
```

TUN mode requires `sudo` on first run. For a proxy-only mode that needs no
privileges:

```bash
vilanet-cli connect --no-tun --mixed 127.0.0.1:1080
```

To install as a per-user LaunchAgent (starts at login, proxy mode):

```bash
sudo bash scripts/install-macos.sh --agent
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.viraltech.vilanet.cli.plist
```

### Windows

```powershell
gh release download --repo vilavpn/vilanet-cli `
    --pattern 'vilanet-cli_*_windows_amd64.zip' --clobber
Expand-Archive -Force vilanet-cli_*_windows_amd64.zip .
```

Run from an **elevated** PowerShell, or install as a Windows Service
(required for background operation — `--background` is not supported on
Windows):

```powershell
New-Item -ItemType Directory -Force "$Env:ProgramFiles\VilaNet" | Out-Null
Copy-Item .\vilanet-cli.exe "$Env:ProgramFiles\VilaNet\vilanet-cli.exe"
.\packaging\windows\install-service.ps1
Start-Service VilaNetCLI
```

---

## Quick start

```bash
vilanet-cli login --email you@example.com    # prompts for password
vilanet-cli list servers --country HK        # browse available nodes
vilanet-cli connect --country HK             # foreground; Ctrl-C to stop
```

Background daemon (Linux/macOS):

```bash
vilanet-cli connect --background --country HK
vilanet-cli status --json                    # check daemon state
vilanet-cli switch --node <hex-id>           # swap node without reconnecting
vilanet-cli disconnect                       # idempotent; safe even if stopped
```

---

## Command reference

| Command | What it does |
|---------|-------------|
| `login [--email E]` | Authenticate; stores credential in OS keyring |
| `list packages` | Show active packages |
| `list servers [--country ISO2]` | Browse available nodes |
| `connect [--auto \| --country ISO2 \| --node ID]` | Start VPN tunnel |
| `connect --background` | Detach to daemon (Linux/macOS only) |
| `connect --no-tun --mixed HOST:PORT` | SOCKS5+HTTP proxy, no TUN/privileges needed |
| `switch --node ID` | Swap node in-place (~1–2 s packet loss) |
| `status [--json] [--stats]` | Daemon state and traffic counters |
| `disconnect` | Stop daemon |
| `settings list` | Dump all settings |
| `settings get KEY` | Read one setting |
| `settings set KEY VALUE` | Persist a setting |
| `logs [--follow]` | Tail the rolling daily log |
| `diag [--output PATH]` | Write a redacted support bundle |
| `logout [--purge]` | Clear keyring and caches |

Global flags: `--json`, `--debug`, `--insecure-store`, `--config DIR`,
`--version`.

---

## Settings (selected)

```bash
vilanet-cli settings set routing_mode global        # bypass China bypass rules
vilanet-cli settings set routing_mode direct        # pause tunnel, all-direct
vilanet-cli settings set tun.stack system           # system / gvisor / mixed
vilanet-cli settings set tun.ipv6 true              # dual-stack TUN
vilanet-cli settings set hy2.speed_mode custom      # Hysteria2 bandwidth hint
vilanet-cli settings set hy2.custom_mbps 200
vilanet-cli settings set hop_interval 10            # Hy2 port-hop seconds
vilanet-cli settings set network.dns_mode fakeip    # fakeip (default) or real
vilanet-cli settings set network.mux_enabled true   # connection multiplexing
vilanet-cli settings set network.sniff_enabled false
vilanet-cli settings set network.block_quic true    # block UDP-443/HTTP3
vilanet-cli settings set network.block_dot true     # block DNS-over-TLS
vilanet-cli settings set network.block_stun true    # block WebRTC STUN
vilanet-cli settings set block_ads true
vilanet-cli settings set block_porn true
```

Config directory by platform:
- **Linux**: `$XDG_CONFIG_HOME/vilanet/` (default `~/.config/vilanet/`)
- **macOS**: `~/Library/Application Support/com.viraltech.vilanet/`
- **Windows**: `%APPDATA%\VilaNet\`

---

## Exit codes

Stable across releases — shell scripts may rely on these.

| Code | Meaning | First step |
|------|---------|------------|
| 0 | Success | — |
| 1 | Generic error (bad flag, I/O failure) | Re-run with `--debug` |
| 2 | Authentication failed | `vilanet-cli login` again |
| 3 | Account closed | Contact VilaVPN support |
| 4 | Rate-limited or no stored credentials | Run `vilanet-cli login` |
| 5 | TUN privileges missing | `sudo setcap` (Linux) / `sudo` (macOS) / run elevated (Windows); or use `--no-tun --mixed` |
| 9 | `--background` not supported on Windows | Install as a Windows Service |

---

## Logs

Rolling per-day logs:
- **Linux**: `${XDG_STATE_HOME:-~/.local/state}/vilanet/logs/`
- **macOS**: `~/Library/Logs/com.viraltech.vilanet/`
- **Windows**: `%LOCALAPPDATA%\VilaNet\Logs\`

Tail live: `vilanet-cli logs --follow`

---

## Use with AI assistants

`ai/vilanet-cli/SKILL.md` is a plain-markdown
[Anthropic Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
that teaches any modern AI coding assistant how to drive the CLI on
your behalf — connect, switch nodes, change settings, diagnose exit codes,
and walk through install/uninstall per platform. The content is identical
everywhere; only the install path differs.

### Claude Code

```bash
mkdir -p ~/.claude/skills/vilanet-cli
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o ~/.claude/skills/vilanet-cli/SKILL.md
```

Or symlink from a local checkout (re-runnable; updates automatically):

```bash
mkdir -p ~/.claude/skills/vilanet-cli
ln -sfn "$PWD/ai/vilanet-cli/SKILL.md" ~/.claude/skills/vilanet-cli/SKILL.md
```

### Codex CLI

```bash
mkdir -p ~/.codex/skills/vilanet-cli
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o ~/.codex/skills/vilanet-cli/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o .cursor/rules/vilanet-cli.mdc
```

### Gemini CLI >= 0.41

```bash
gemini skills install https://github.com/vilavpn/vilanet-cli.git --path ai/vilanet-cli
# Or from a local checkout:
gemini skills link ./ai/vilanet-cli
```

Older Gemini CLI builds without native `skills` support use the memory-import
processor. Download locally, then append to `GEMINI.md`:

```bash
mkdir -p .gemini
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o .gemini/vilanet-cli.SKILL.md
grep -qxF '@./.gemini/vilanet-cli.SKILL.md' GEMINI.md 2>/dev/null || \
  printf '\n@./.gemini/vilanet-cli.SKILL.md\n' >> GEMINI.md
```

### Cline, Continue, Aider, Copilot CLI, and any other AI

Universal fallback — writes the skill to a neutral path, then appends to
`AGENTS.md` if needed (idempotent; re-running never duplicates):

```bash
mkdir -p .agents/skills/vilanet-cli
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o .agents/skills/vilanet-cli/SKILL.md

grep -qxF '## VilaNet CLI Agent Skill' AGENTS.md 2>/dev/null || \
  { echo; echo '## VilaNet CLI Agent Skill'; cat .agents/skills/vilanet-cli/SKILL.md; } >> AGENTS.md
```

Once installed, ask your AI to connect, switch nodes, change a setting, or
diagnose an exit code — it will reach for `vilanet-cli`.

---

## Sibling clients

| Client | Where to get it |
|--------|----------------|
| VilaNet for OpenWrt (LuCI + UCI) | [vilavpn/vilanet-openwrt](https://github.com/vilavpn/vilanet-openwrt) |
| VilaNet desktop & mobile (iOS, Android, macOS, Windows GUI) | [vilavpn.com/download](https://vilavpn.com/download) |
| VilaNet for Apple TV | [App Store](https://vilavpn.com) |

---

## Links

- User guide: <https://cli.vilavpn.com>
- VilaVPN: <https://vilavpn.com>
- Releases: <https://github.com/vilavpn/vilanet-cli/releases>
- Skill file (raw, CDN-cached): <https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md>
- OpenWrt sibling: <https://github.com/vilavpn/vilanet-openwrt>

---

## License

See `LICENSE` in the repository root.
