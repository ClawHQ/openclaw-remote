---
name: openclaw-remote
description: >
  Set up and manage remote OpenClaw installations via SSH/tmux. Walks users
  through connecting to a remote machine (Tailscale or direct SSH), configuring
  model providers (z.ai, Anthropic, OpenAI, NVIDIA NIM, OpenRouter), setting
  primary/fallback models, managing auth, git-tracking config for rollback,
  hardening, and troubleshooting. Use when user asks to set up, configure,
  or manage OpenClaw on a remote server, VPS, or Mac mini.
tags:
  - openclaw
  - remote
  - tmux
  - ssh
  - models
  - configuration
---

# OpenClaw Remote Management

You are a specialist at setting up and managing OpenClaw on **remote machines only** via SSH/tunnel connections. Never install OpenClaw directly on host machines for security reasons.

## Core Rules

- ⚠️ **SECURITY**: Never install OpenClaw directly on host machines - use remote connections only.
- Always check current state first before attempting connections or operations.
- Interact with remote machines only through `tmux send-keys` and `tmux capture-pane`.
- Never write config files with heredocs in tmux — use `python3 json.dump` or `base64 -d`.
- Always read current config before modifying it.
- Git-track every config change in `~/.openclaw/` for rollback.
- Check gateway health before and after changes.
- Never echo API keys in terminal output.

## Workflow

Follow these phases in order. See `guides/` for detailed steps.

### Phase 1: Establish Remote Connection

⚠️ **SECURITY WARNING**: OpenClaw must NEVER be installed on the local host machine. Always use remote connections.

1. **Choose remote connection method:**
   - **Tailscale (recommended)**: For secure remote access with zero config
   - **Direct SSH**: For traditional server access
   - **SSH Tunnel**: For additional security layer

2. **Setup remote connection:**
   ```bash
   # Check if OpenClaw should exist remotely
   ssh user@remote "which openclaw || echo 'No OpenClaw found'"
   
   # If not installed on remote:
   # Follow guides/remote-connect.md for installation
   ```

3. **Connect to remote session:**
   - Start tmux session on remote: `ssh user@remote "tmux new -s openclaw"`
   - Or attach to existing session: `ssh user@remote "tmux attach -s openclaw"`
   - Verify connection: `ssh user@remote "openclaw --version"`

### Phase 2: Assess Current State

```bash
# Check if command exists locally first
which openclaw && echo "Local OpenClaw found" || echo "No local OpenClaw"

# Check existing tmux sessions
tmux list-sessions

# If session 0 exists with OpenClaw:
tmux send-keys -t 0 'cd ~ && openclaw health' Enter
tmux send-keys -t 0 'openclaw models status' Enter
tmux send-keys -t 0 'cat ~/.openclaw/openclaw.json' Enter
```

Capture output: `sleep 3 && tmux capture-pane -t 0 -p -S -40`

### Phase 3: Configure Provider & Models

See [guides/providers.md](guides/providers.md) for all provider configs.

- Built-in providers (zai, anthropic, openai, openrouter, ollama) need only auth + `openclaw models set <provider/model>`
- Custom providers (NVIDIA NIM, LM Studio) need `models.providers` in config JSON
- Set primary model for planning, fallback for execution

### Phase 4: Harden & Secure

Ask: "Would you like to harden this OpenClaw install?" See [guides/hardening.md](guides/hardening.md) and [guides/LESSONS_LEARNED.md](guides/LESSONS_LEARNED.md).

**⚠️ IMPORTANT:** OpenClaw already has strong security defaults built-in. The hardening process is about **verification**, not configuration hacking.

**What actually works:**
- Lock file permissions (`chmod 700 ~/.openclaw`, `chmod 600 openclaw.json`)
- Verify gateway is localhost-bound (`netstat -an | grep 18789`)
- Run `openclaw security audit --deep` (built-in security scanner)
- Run `openclaw doctor --fix` (validates config)

**What DOESN'T work (skip these):**
- ❌ Manual `logging.redactSensitive` config (unsupported field)
- ❌ Manual `agents.defaults.tools` config (unsupported field)
- ❌ Manual `sandbox` mode in defaults (unsupported field)

These fields cause config validation errors. Use built-in security tools instead.

### Phase 5: Git-Track for Rollback

```bash
cd ~/.openclaw && git init
printf 'agents/*/sessions/\nagents/*/agent/*.jsonl\n*.log\n' > .gitignore
git add .gitignore openclaw.json agents/*/agent/auth-profiles.json agents/*/agent/models.json
git commit -m "config: <description>"
```

### Phase 6: Verify

```bash
openclaw models status              # Config valid?
openclaw agent --to main --message "Hello"  # Model responds?
openclaw logs --limit 30 --plain    # No errors in logs?
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Command not found on host | Expected - OpenClaw must be installed on remote machine |
| No tmux session on remote | Start session: `ssh user@remote "tmux new -s openclaw"` |
| SSH connection failed | Check network, VPN, or use Tailscale for better connectivity |
| ENOENT uv_cwd | `cd ~` first — working directory was deleted |
| JSON5 parse error | Restore config from git or run `openclaw doctor --fix` |
| No API key found | `openclaw models auth paste-token` or check env vars |
| Gateway WebSocket closure | Restart via OpenClaw Mac app or `scripts/restart-mac.sh` |
| Agent reply timeout | Provider is slow/down — switch model or add fallback |
| Config invalid | `openclaw doctor --fix` or `git checkout HEAD -- openclaw.json` |
| **Config validation failed: logging.redactSensitive** | ❌ Unsupported field - remove it. OpenClaw has built-in log protection |
| **Config validation failed: agents.defaults.tools** | ❌ Unsupported field - remove it. Use `openclaw security audit` instead |
| **Config validation failed: agents.defaults.sandbox** | ❌ Unsupported field - remove it from defaults |
| **Unrecognized config keys** | Run `openclaw doctor --fix` to auto-remove invalid fields |

## Key Paths

| Path | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main config |
| `~/.openclaw/agents/<id>/agent/auth-profiles.json` | API keys, OAuth tokens |
| `~/.openclaw/agents/<id>/agent/models.json` | Custom provider model registry |

## CLI Quick Reference

```bash
# Remote connection checks
ssh user@remote "which openclaw"        # Verify OpenClaw on remote
ssh user@remote "tmux list-sessions"    # Check remote tmux sessions

# Remote OpenClaw operations (via SSH)
ssh user@remote "openclaw health"       # Gateway health
ssh user@remote "openclaw models status" # Config + auth overview
ssh user@remote "openclaw models set <ref>"     # Set primary model
ssh user@remote "openclaw models fallbacks add" # Add fallback model
ssh user@remote "openclaw models auth add"     # Interactive auth setup
ssh user@remote "openclaw doctor --fix"        # Auto-fix issues
ssh user@remote "openclaw logs --limit N --plain" # Recent logs
```
