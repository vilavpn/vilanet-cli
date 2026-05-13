# vilanet-cli — public release & skill mirror

This repository is the **public distribution surface** for
[`vilanet-cli`](https://vilavpn.com), the cross-platform command-line
VilaNet VPN client. It holds:

- **Release binaries** for Linux, macOS, and Windows (under
  [Releases](https://github.com/vilavpn/vilanet-cli/releases)).
- **The AI Agent Skill** at
  [`ai/vilanet-cli/SKILL.md`](ai/vilanet-cli/SKILL.md), which teaches
  Claude Code, Codex CLI, Gemini CLI, Cursor, Cline, Aider, Copilot
  CLI, and any other modern AI assistant how to drive the CLI on
  behalf of an end user.

The source code lives in a separate, private repository. This mirror
is for distribution only.

---

## Install the CLI

Pre-built binaries for the latest release are versioned. The simplest
fetch uses the GitHub CLI with a wildcard pattern:

### Linux

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_linux_amd64.tar.gz' --clobber
tar -xzf vilanet-cli_*_linux_amd64.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
sudo setcap 'cap_net_admin,cap_net_bind_service+eip' /usr/local/bin/vilanet-cli
```

### macOS

```bash
gh release download --repo vilavpn/vilanet-cli \
    --pattern 'vilanet-cli_*_darwin_arm64.tar.gz' --clobber   # or _darwin_amd64 on Intel
tar -xzf vilanet-cli_*_darwin_*.tar.gz
sudo install -m 0755 vilanet-cli /usr/local/bin/
```

### Windows

```powershell
gh release download --repo vilavpn/vilanet-cli `
    --pattern 'vilanet-cli_*_windows_amd64.zip' --clobber
Expand-Archive -Force vilanet-cli_*_windows_amd64.zip .
```

Then run elevated, or install as a Windows Service.

For the full install / settings / troubleshooting guide, see the
[interactive user guide](https://vilavpn.com).

---

## Use with AI assistants

`ai/vilanet-cli/SKILL.md` is a plain-markdown
[Anthropic Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
— any modern AI tool can consume it. The content is identical
everywhere; only the install path differs.

### Claude Code

```bash
mkdir -p ~/.claude/skills/vilanet-cli
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o ~/.claude/skills/vilanet-cli/SKILL.md
```

### Gemini CLI ≥ 0.41

```bash
gemini skills install https://github.com/vilavpn/vilanet-cli.git --path ai/vilanet-cli
# Or from a local checkout:
gemini skills link ./ai/vilanet-cli
```

### Codex CLI · Cursor · Cline · Aider · Copilot CLI · anything else

```bash
mkdir -p .agents/skills/vilanet-cli
curl -fsSL https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md \
  -o .agents/skills/vilanet-cli/SKILL.md

# For tools that only read AGENTS.md, append (idempotent — re-runs are a no-op):
grep -qxF '## VilaNet CLI Agent Skill' AGENTS.md 2>/dev/null || \
  { echo; echo '## VilaNet CLI Agent Skill'; cat .agents/skills/vilanet-cli/SKILL.md; } >> AGENTS.md
```

Once installed, ask your AI to connect, switch nodes, change a
setting, or diagnose an exit code — it will reach for `vilanet-cli`.

---

## Links

- VilaVPN: <https://vilavpn.com>
- Releases: <https://github.com/vilavpn/vilanet-cli/releases>
- Skill file (raw, CDN-cached): <https://testingcf.jsdelivr.net/gh/vilavpn/vilanet-cli@main/ai/vilanet-cli/SKILL.md>
