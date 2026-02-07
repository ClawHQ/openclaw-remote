# Remote Connection Guide

## Step 1: Determine Connection Method

Ask the user:
> How do you connect to your remote machine?
> 1. Tailscale (recommended — zero-config mesh VPN)
> 2. Direct SSH to a VPS (public IP)
> 3. Local network (same LAN)

## Step 2: Connect via Tailscale

If the user has Tailscale installed on both machines:

```bash
# Check Tailscale is running
tailscale status

# SSH to remote using Tailscale hostname
ssh <user>@<hostname>.tail<tailnet>.ts.net
```

If Tailscale is not installed:

```bash
# macOS
brew install tailscale

# Linux
curl -fsSL https://tailscale.com/install.sh | sh

# Start and authenticate
sudo tailscale up
```

Then on the remote machine, also install and authenticate Tailscale.

## Step 2 (alt): Connect via Direct SSH

```bash
# Test connection
ssh <user>@<ip-address>

# If password-based, recommend switching to key-based auth
```

## Step 3: Set Up SSH Key Authentication (if needed)

If the user authenticates with a password, walk them through key-based auth:

```bash
# Generate key (on local machine)
ssh-keygen -t ed25519 -C "openclaw-remote"

# Copy to remote
ssh-copy-id <user>@<remote-address>

# Test passwordless login
ssh <user>@<remote-address> 'echo "Key auth works"'
```

For Windows/PowerShell users:

```powershell
# Generate key
ssh-keygen -t ed25519 -C "openclaw-remote"

# Copy key manually (no ssh-copy-id on Windows)
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <user>@<remote-address> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## Step 4: Start tmux Session on Remote

```bash
# SSH in and start tmux
ssh <user>@<remote-address>
tmux new-session -s openclaw

# Or attach to existing
ssh <user>@<remote-address> -t 'tmux attach-session -t openclaw || tmux new-session -s openclaw'
```

## Step 5: Use tmux from Local Agent

Once the user has an SSH connection, interact via tmux from the local machine:

```bash
# If SSH session has tmux running locally that forwards to remote:
tmux send-keys -t <local-session> 'openclaw --version' Enter
sleep 2 && tmux capture-pane -t <local-session> -p -S -5

# If using SSH directly in tmux:
tmux send-keys -t <session> 'ssh <user>@<remote> "openclaw --version"' Enter
```

## Step 6: Verify OpenClaw Installation

```bash
which openclaw && openclaw --version
```

If not installed:

```bash
# macOS (Homebrew)
brew install openclaw

# Linux (npm)
npm install -g openclaw

# Verify
openclaw --version
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Connection refused | Check SSH is running: `sudo systemctl status sshd` |
| Permission denied | Check key permissions: `chmod 600 ~/.ssh/id_ed25519` |
| Tailscale not connecting | Run `tailscale up --reset` on both machines |
| tmux not found | Install: `brew install tmux` (mac) or `apt install tmux` (linux) |
| ENOENT uv_cwd in tmux | Run `cd ~` first — previous cwd was deleted |
