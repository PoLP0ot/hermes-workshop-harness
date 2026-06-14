# Hermes Harness Workshop

A ready-to-run Hermes Agent profile for AI-assisted full-stack app generation using the **Harness Workflow**: Claude Code (design + code) + Open Design (visual) + Hermes (PM + QA gate).

## What's included

- **`harness-workflow` skill** — the universal project pipeline: PM decomposes into user stories → Claude Code `/plan` → `/code` in TDD → Hermes QA gate
- **`claude-code` / `claude-code-low-latency` skills** — delegate all code to Claude Code via tmux (interactive mode, NEVER `claude -p`)
- **`open-design` skill + MCP server config** — AI-driven high-fidelity design mockups
- **20+ supporting skills** — TDD, code review, debugging, architecture diagrams, GitHub PR workflow, etc.
- **Memory pre-loaded** with Docker/uv workflow conventions and tmux patterns
- **Clean config** — no personal data, no API keys, ready to customize

## Prerequisites

### 1. Hermes Agent

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

### 2. External tools (install once)

**Claude Code** (Anthropic's coding agent):
```bash
npm install -g @anthropic-ai/claude-code
# Authenticate: claude
```

**Node.js 24** (required for Open Design MCP server):
```bash
# Download and extract
curl -fsSL https://nodejs.org/dist/v24.16.0/node-v24.16.0-linux-x64.tar.xz | tar xJ -C /opt/data/
```

**Open Design** (AI design tool, MCP server):
```bash
git clone https://github.com/nexu-io/open-design.git /opt/data/open-design
cd /opt/data/open-design
npm install
npm run build
# Start the daemon (keep running in background)
node apps/daemon/dist/cli.js daemon &
```

**tmux** (required for Claude Code interactive mode):
```bash
# Debian/Ubuntu
apt-get install -y tmux
# macOS
brew install tmux
```

## Quick Start

### Import the profile

```bash
hermes profile import hermes-workshop.tar.gz
```

### Configure your API keys

Edit `~/.hermes/profiles/workshop/.env`:

```bash
# Required — your LLM provider
DEEPSEEK_API_KEY=sk-...
# or
ANTHROPIC_API_KEY=sk-...
# or
OPENROUTER_API_KEY=sk-...

# Edit the model in config if not using DeepSeek:
# hermes --profile workshop config set model.provider openrouter
# hermes --profile workshop config set model.default anthropic/claude-sonnet-4
```

### Launch

```bash
# Start with the workshop profile
hermes --profile workshop

# Or use the wrapper (if ~/.local/bin is in PATH)
workshop
```

## Workflow Overview

```
IDEA
 │
 ├─ HERMES (PM): writes project brief + business logic (non-technical)
 │
 ├─ CLAUDE CODE + OPEN DESIGN: vibe → mockups → DESIGN.md
 │    → Human validates design (2 checks)
 │
 ├─ HERMES: github repo + harness bootstrap (hermes-project-init)
 │
 ├─ CLAUDE CODE (/plan): architecture → CLAUDE.md Stack (contract)
 │    → Human validates stack (1 check)
 │
 ├─ GO/NO-GO → if GO, human leaves
 │
 └─ AUTO MODE:
      ├─ PHASE 0: Claude bootstraps project (docker-compose, deps, test harness)
      ├─ HERMES: decomposes into non-technical user stories
      └─ PHASES 1→N: /plan → /code (TDD) → QA gate → next US
           └─ HERMES pings human when ALL stories done + QA verified
```

## Key Rules

1. **NEVER `claude -p`** — always tmux interactive mode (the harness context lives in `.claude/`)
2. **Hermes never codes** — all code, design, and architecture decisions come from Claude
3. **TDD always** — RED → GREEN → REFACTOR, no exceptions
4. **Visual QA is mandatory** — tests passing ≠ app works

## Files

```
hermes-workshop.tar.gz    # Profile archive (import via `hermes profile import`)
README.md                 # This file
```

## Troubleshooting

**MCP open-design fails**: ensure the daemon is running (`node apps/daemon/dist/cli.js daemon &`) and Node.js is at `/opt/data/node-v24.16.0-linux-x64/bin/node`. Adjust paths in `config.yaml` → `mcp_servers.open-design` if needed.

**uv permission errors in Docker**: `chown -R hermes:hermes /opt/hermes/.venv/` from root, then use `uv pip install` (never plain `pip`).

**`hermes-project-init` fails**: ensure `gh` CLI is installed and authenticated.

## License

MIT — use freely for workshops, teams, and projects.
