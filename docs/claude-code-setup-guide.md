# Claude Code Setup & Integration Guide

## Prerequisites

- Claude Code installed: `npm install -g @anthropic-ai/claude-code`
- tmux installed (apt-get or static binary)
- Node.js 24 for Open Design (optional, separate concern)

## Environment

```bash
export HOME=/opt/data/home          # Claude stores config here
export CLAUDE_HOME=/opt/data/home
```

Create if needed: `mkdir -p /opt/data/home`

---

## Phase 1: Authentication (One-Time)

### The OAuth `#` Trap (CRITICAL)

OAuth codes from `claude auth login` **contain literal `#` characters** as part of the code value. This is the #1 integration failure point.

**The `#` is NOT a separator between code and state.** The entire string including `#` is the code.

Example code from callback URL:
`TP7coauRnozcldLTNQgkT75cJRapX6OkoheElyxnUhk8hAya#coioqSCmFlqC-MRKl7-d0SCQqWO17_fH85p4v0IT8iE`

Sending only the part before `#` (`TP7coauRnozcldLTNQgkT75cJRapX6OkoheElyxnUhk8hAya`) → **"Invalid code" every time.**

### PKCE Session Binding (CRITICAL)

Every invocation of `claude auth login` generates a NEW PKCE challenge (code_challenge + state). The OAuth code returned by the browser is bound to ONE specific session. If the `claude auth login` process dies before receiving the code, the code becomes worthless.

**Rule: capture the URL, send it to the user, receive the code, and feed it back — all within the same `claude auth login` process lifetime.**

### Method 1: Interactive tmux (recommended for headless)

```bash
# 1. Start Claude in tmux
tmux new-session -d -s claude -x 200 -y 40
tmux send-keys -t claude "HOME=/opt/data/home claude" Enter

# 2. Wait for theme prompt (~4s), accept default
sleep 4
tmux send-keys -t claude Enter

# 3. Wait for workspace trust (~3s), accept
sleep 3
tmux send-keys -t claude Enter

# 4. Type /login via buffer method
echo -n '/login' > /tmp/login.txt
tmux load-buffer /tmp/login.txt
tmux paste-buffer -t claude
tmux send-keys -t claude Enter
sleep 3

# 5. Capture the OAuth URL from the pane
tmux capture-pane -t claude -p -J -S -20 | grep -o 'https://claude.com/cai/oauth/authorize[^ ]*'

# 6. Send URL to user → user opens it → user sends back the FULL code (# included)

# 7. Paste the FULL code via BUFFER METHOD (NEVER send-keys with code inline!)
echo -n 'FULL_CODE_WITH_#_AND_EVERYTHING_AFTER' > /tmp/oauth_code.txt
tmux load-buffer /tmp/oauth_code.txt
tmux paste-buffer -t claude
sleep 0.3
tmux send-keys -t claude Enter

# 8. Wait for "Login successful" (~5s)
sleep 5

# 9. Press Enter to complete
tmux send-keys -t claude Enter
```

### Method 2: `claude auth login --console` (alternative)

Same principle, but command is `claude auth login --console` instead of interactive Claude + `/login`. The URL will be from `platform.claude.com` instead of `claude.com/cai`. Same buffer method for the code.

### Verification

```bash
HOME=/opt/data/home claude auth status
# → {"loggedIn": true, "authMethod": "claude.ai", "email": "..."}

HOME=/opt/data/home claude -p 'Reply exactly: OK' --max-turns 1 --output-format json
# → {"result":"OK",...}
```

---

## Phase 2: Harness Installation (Per Project)

The claude-harness provides `.claude/skills/`, `rules/`, `agents/`, `hooks/`, and state signaling.

```bash
git clone https://github.com/PoLP0ot/claude-harness.git /tmp/claude-harness
/tmp/claude-harness/scripts/install.sh /path/to/your-project
```

This deploys:
- 17 skills, 12 rules, 7 agents
- Security hooks (block dangerous commands, block secret access)
- State signaling via `/tmp/claude-state.json`
- Permission settings (allow git/npm/docker, deny .env/sudo/rm -rf)

---

## Phase 3: Launching Claude Code (tmux)

### Full Launch Sequence

```bash
PROJECT_DIR="/opt/data/projects/my-app"
SESSION="claude"

# 1. Create session
tmux new-session -d -s "$SESSION" -x 200 -y 40

# 2. Launch Claude with bypass permissions (BUFFER METHOD)
echo -n "cd $PROJECT_DIR && HOME=/opt/data/home claude --dangerously-skip-permissions" > /tmp/launch.txt
tmux load-buffer /tmp/launch.txt
tmux paste-buffer -t "$SESSION"
sleep 0.3
tmux send-keys -t "$SESSION" Enter

# 3. Handle startup dialogs
sleep 6
tmux send-keys -t "$SESSION" Enter    # Trust folder (default: Yes)

sleep 3
tmux send-keys -t "$SESSION" Down     # Bypass permissions → change from default "No"
sleep 0.3
tmux send-keys -t "$SESSION" Enter    # "Yes, I accept"

# 4. Verify status bar shows "⏵⏵ bypass permissions on"
sleep 2
tmux capture-pane -t "$SESSION" -p -S -10 | grep "bypass"

# 5. Send task (BUFFER METHOD)
echo -n '/plan Analyze the user story. Plan only.' > /tmp/cmd.txt
tmux load-buffer /tmp/cmd.txt
tmux paste-buffer -t "$SESSION"
sleep 0.3
tmux send-keys -t "$SESSION" Enter

# 6. Wait for Claude to finish (~15s initialization + processing time)
# Check state file:
cat /tmp/claude-state.json
# → {"status":"idle","ts":...}
```

---

## Phase 4: Monitoring & State Signaling

### State File (preferred)

The harness writes `/tmp/claude-state.json`:

```bash
cat /tmp/claude-state.json
# {"status":"working"|"idle"|"prompt","ts":<unix_ms>}
```

- `working` = Claude is executing (PreToolUse hook fires)
- `idle` = Claude is waiting for next prompt (Stop hook fires)
- `prompt` = Claude needs user input (Notification hook fires for permission dialogs)

**Poll state file instead of capture-pane for efficiency.**

### Capture Pane (fallback, use sparingly)

```bash
tmux capture-pane -t claude -p -J -S -30
```

Only use when you need to read Claude's output, not for "is it done?" checks.

---

## Pitfalls Checklist

| # | Pitfall | Fix |
|---|---------|-----|
| 1 | OAuth code truncated at `#` | Buffer method — `echo -n` → `load-buffer` → `paste-buffer` |
| 2 | PKCE session mismatch | Keep `claude auth login` process alive until code received |
| 3 | `send-keys` queues text, never executes | Always use buffer method for commands |
| 4 | Paste-buffer lost during permission dialog | Accept/dismiss dialog first, wait 2s, then paste |
| 5 | Claude Code not finding config | Always set `HOME=/opt/data/home` |
| 6 | Missing `--dangerously-skip-permissions` | Flag must be passed at launch; two dialogs to accept |
| 7 | `claude -p` drops harness context | Always tmux interactive mode, never `-p` |
| 8 | Tmux initialization delay (15-20s) | Wait for `❯` prompt before first command |
| 9 | State file not written | Harness not installed → run `install.sh` |
| 10 | Permission prompts blocking AUTO mode | Use `--dangerously-skip-permissions` or auto-accept script |

---

## Quick Reference Card

```bash
# Auth
HOME=/opt/data/home claude auth status

# Smoke test
HOME=/opt/data/home claude -p 'OK' --max-turns 1

# Start tmux session
tmux new-session -d -s claude -x 200 -y 40

# Send command (NEVER raw send-keys)
echo -n 'command here' > /tmp/cmd.txt
tmux load-buffer /tmp/cmd.txt && tmux paste-buffer -t claude
sleep 0.3 && tmux send-keys -t claude Enter

# Check state
cat /tmp/claude-state.json

# Read output
tmux capture-pane -t claude -p -J -S -30

# Kill session
tmux kill-session -t claude
```
