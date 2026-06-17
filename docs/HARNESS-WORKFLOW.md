# Hermes Workshop Harness — Complete Workflow

**Version:** 2.0 | **Last updated:** 2026-06-17
**Purpose:** End-to-end pipeline for building production-grade web apps with Hermes (PM/QA) + Claude Code (engineer) + OpenDesign (visual design)

---

## 1. Roles & Responsibilities

| Role | Who | Responsibilities |
|------|-----|-----------------|
| **PM** | Hermes | Product intent, scope, prioritization, competitor research, approvals, QA gate |
| **Engineer** | Claude Code | All code changes, implementation, bug fixes, tests, architecture decisions |
| **Designer** | OpenDesign MCP (via Claude) | Vibe exploration, hi-fi mockups, screen maps, visual design |

### Iron Rules

- Hermes **never writes code** — Claude does all coding
- Claude runs in **tmux interactive mode** (NOT `claude -p` print mode — it drops the harness context)
- Every design task MUST use **OpenDesign MCP**
- All Claude launches use `--dangerously-skip-permissions`
- Post-Claude verification is **mandatory** after every session (5 checks)
- Communication with Claude: **buffer method** (`load-buffer` + `paste-buffer`), NEVER `send-keys` for content with special characters
- Hermes validates Claude's plans autonomously — only reports to human when ALL user stories are done + QA verified

---

## 2. Workflow Overview

```
SPECS → DESIGN → INIT → /PLAN → GO → IMPLEMENTATION → DATA SEED → QA → BUG FIX → DELIVER
```

9 phases total. Each produces specific deliverables and passes defined verification gates.

---

## 3. Phase 1: Specs & Research (Hermes as PM)

### Steps

1. Capture user request verbatim
2. Ask clarifying questions via `clarify()` — audience, differentiator, must-haves, tech constraints
3. Research competitors via `web_search` (find 3-5, identify gaps and differentiation opportunities)
4. Write refined brief → `.claude/memory/project-brief.md` (elevator pitch + P1/P2/P3 features)
5. Write business logic → `.claude/memory/business-logic.md` (data model + business rules)

### Deliverables

- `project-brief.md` — what are we building, for whom, why it's different
- `business-logic.md` — entities, relationships, rules, edge cases

---

## 4. Phase 2: Design (Claude + OpenDesign MCP)

### Steps

1. **Vibe exploration:** Claude generates 3 distinct visual vibes via OpenDesign MCP (Bold/Arcade, Premium/Luxury, Minimal/Gallery)
2. **User picks one** vibe direction
3. **Screen map:** Claude creates flowchart with clickable SVG nodes via MCP — shows every page and navigation flow
4. **Hi-fi mockups:** Claude builds mockups for ALL screens via MCP (search, detail, collection, dashboard, settings — every page in the screen map)
5. **Post-Claude verification** (5 checks):
   - Were MCP `mcp__open_design_*` calls actually made? (check Claude session logs)
   - Do all internal links work? (open every mockup, click every link)
   - Any blurs, sheens, or screen-blend effects? (remove — standard is `scale(1.02)` + border-color only)
   - All interactive elements functional? (buttons, inputs, selects)
   - Pointer-events safe? (no dead zones, no hidden overlays eating clicks)

### Deliverables

- 3 vibe HTMLs
- 1 screen map
- 4-6 mockup HTMLs (one per screen)

### Delivery

Always share via Cloudflare tunnel so stakeholders can click through:
```bash
npx cloudflared tunnel --url http://localhost:3000 &
```

---

## 5. Phase 3: Init & Scaffold

### Steps

1. `mkdir -p /opt/data/projects/{project}` + `git init`
2. Create directory structure: `design/`, `src/`, `data/`, `.hermes/`, `.claude/`
3. Run `hermes-project-init` — installs CLAUDE.md, memory, skills, rules, agents, hooks, state signaling
4. Configure stop hook (`notify-state.sh` writes `/tmp/claude-state.json` on completion)
5. Create `wait-claude.py` to monitor async completion (polls state file)

### Verification

```bash
cat /tmp/claude-state.json
# → {"status":"idle","ts":...}

ls .claude/skills/ .claude/rules/ .claude/agents/ .claude/hooks/
# → all directories populated
```

### Deliverable

Full harness with stop hook for async monitoring (`claude-state.json`).

---

## 6. Phase 4: /plan (Architecture Contract)

### Steps

1. Send Claude: "Read ALL `.claude/memory/*.md` files. Read the mockup designs. Fill the Stack section of CLAUDE.md."
2. Claude reads project brief + business logic + mockups
3. Claude proposes concrete Stack: runtime, framework, database, CSS, component library, APIs, data layer, deployment, testing framework
4. **Conflict detection:** Claude identifies conflicts between mockup styling and chosen frameworks (e.g., hand-authored CSS vs ShadCN/Radix)
5. **User resolves** conflicts and approves Stack
6. Claude writes final Stack to `CLAUDE.md`

### Deliverable

`CLAUDE.md` with complete, locked Stack section — this is the architecture contract. No deviations without re-/plan.

---

## 7. Phase 5: GO → AUTO Implementation

### Trigger

User says "GO". Hermes dispatches to Claude and monitors. From this point on, Hermes operates autonomously — no human intervention until all stories are done.

### Steps

1. Claude defines **7-8 user stories** with acceptance criteria (decompose the brief into testable units)
2. Claude implements each story sequentially via hook-chaining:
   - Each story completion triggers the next via `notify-state.sh`
   - TDD enforced: write test first → RED → GREEN → REFACTOR → BUILD
3. Hermes monitors via `wait-claude.py` + `/tmp/claude-state.json` — non-polling progress detection
4. After each story, Hermes verifies:
   - All tests pass: `npx vitest run`
   - Build clean: `npx next build`
   - ESLint zero: `npx eslint src/`
   - i18n key diff zero: no missing keys across locales

### Per-story acceptance criteria

| State | What to verify |
|-------|---------------|
| **Loading** | Skeleton/spinner present, no flash of empty state |
| **Empty** | Empty state message, CTA to add data |
| **Data** | Data renders correctly, interactions work |
| **Error** | Error boundary catches, user-friendly message |
| **404** | Not-found page renders for invalid routes |

### Deliverable

Full app with:
- 90+ passing tests (Vitest + React Testing Library)
- All screens functional
- i18n complete across all configured locales
- Loading, empty, error, 404 states handled on every page

---

## 8. Phase 6: Data Seeding

### Steps

1. Define a persona (e.g., "impulsive collector", "casual hobbyist", "power seller")
2. Generate realistic seed data:
   - 30-40 items with varied attributes
   - Realistic pricing (not all round numbers)
   - Purchase bursts (multiple items same day)
   - Some overpays, some bargains
   - Bulk duplicates of commons
3. Save as `data/collection.json`
4. Verify dashboard shows realistic metrics (P&L, distribution, trends)

### Deliverable

`data/collection.json` — realistic seed data that exercises all dashboard/analytics paths.

---

## 9. Phase 7: QA — Documentation-Driven Visual Testing

This is the most detailed phase. QA is **documentation-driven**: every check traces back to a line in `QA-SPEC.md`.

### Step 7.1: Build the QA Specification

Claude generates `QA-SPEC.md` — an exhaustive reference document listing every interactive element on every screen (60-70 items across all pages). Each entry has:
- Element identifier (e.g., "Search input", "Type filter dropdown", "Add to Collection button")
- Trigger action (click, type, select, hover, scroll)
- Expected result (what should happen)
- States covered: success, error, empty, loading
- Cross-cutting concerns: i18n (all locales), responsive (desktop/tablet/mobile), hover effects, accessibility

### Step 7.2: Start the Application Under Test

```bash
# Start dev server
cd /path/to/project && npm run dev &

# Wait for readiness
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/en

# Create Cloudflare tunnel for visual inspection
npx cloudflared tunnel --url http://localhost:3000 &
```

### Step 7.3: Install Browser Engine (first time)

```bash
npx agent-browser install
mkdir -p /opt/data/.agent-browser/browsers
ln -sf /opt/data/home/.agent-browser/browsers/chrome-* /opt/data/.agent-browser/browsers/
```

### Step 7.4: Systematic Visual QA Against Spec

For **every QA-SPEC item**, across **all locales**:

1. **Server render check** — `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/{locale}/{page}`
2. **Accessibility tree check** — `browser_snapshot(full=true)` to verify all expected elements exist with correct roles/labels
3. **Visual check** — `browser_vision` with specific questions about layout, spacing, color, overlap
4. **Console check** — `browser_console` to catch JS errors, 400/500 API responses, hydration mismatches
5. **Interaction check** — `browser_click` → `browser_snapshot` to simulate real user flows
6. **API curl verification** — confirm backend correctness independently of frontend

### Step 7.5: Visual Integrity Cross-Page Audit

Check every page for:
- **Overlaps** — do elements cover each other?
- **Overflows** — does content escape its container?
- **Cut-offs** — is any text or element clipped?
- **Alignment** — are cards, buttons, prices aligned in grids?
- **Z-order** — are modals, toasts, tooltips above everything else?
- **Pointer-events** — are all click targets reachable? (no invisible overlays eating clicks)
- **Responsive breakpoints** — mobile hamburger menu works, tablet grid adapts
- **Hover effects** — `scale(1.02)` + `border-color` only; no glows, sheens, shadows, screen-blend

**Rule:** Any bug found on one page triggers a **full sweep** of all pages sharing that component.

### Step 7.6: Bug Triage & Ticketing

Bugs are classified by severity:

| Severity | Criteria | Examples |
|----------|---------|---------|
| 🔴 CRITICAL | Page crashes, no data, navigation broken | React error boundary, zero API calls, build failure |
| 🟡 MEDIUM | Visual/UX defect, missing feedback, broken API param | Missing toast, concatenated labels, search returns 400 |
| 🟢 LOW | Cosmetic, rounding errors, stray characters | Hardcoded emoji, 99% instead of 100% |

Each bug gets a ticket:
```
BUG-XXX — SEVERITY: Title
Source: file path + line numbers
Symptom: exact observed behavior
Expected: what should happen
Root cause hypothesis
Fix checklist
Acceptance criteria
```

Tickets saved to `.hermes/qa-tickets-YYYY-MM-DD.md`.

### Deliverables

- `QA-SPEC.md` — 863-line reference document
- `.hermes/qa-tickets-{date}.md` — numbered bug tickets with full context

---

## 10. Phase 8: Bug Fixing Loop

### Steps

1. Prioritize: CRITICAL first → MEDIUM → LOW
2. Dispatch tickets to Claude via tmux buffer method:
   ```bash
   cat /path/to/tickets.md | tmux load-buffer -
   tmux paste-buffer -t claude
   tmux send-keys -t claude Enter
   ```
3. **Critical instruction in prompt:** "ONLY source code. NEVER edit memory, docs, or design files."
4. Monitor via `/tmp/claude-state.json`
5. Re-run QA to verify each fix (browser navigate + vision + console)
6. Re-dispatch remaining items if partial

### Pitfalls

- Claude sometimes claims bugs are "already fixed" — verify with file mtimes + browser
- Claude sometimes edits `CLAUDE.md` or memory instead of source — explicit directive prevents this
- After `.next/` cache clear, dev server must be restarted
- Fix verification must use **full reload** (not cached page)

### Deliverable

All bugs resolved or explicitly documented with justification.

---

## 11. Phase 9: Delivery & Sign-off

### Final Checks

```bash
npx vitest run          # All passing (90+ tests)
npx next build          # Clean build, zero warnings
npx eslint src/         # Zero issues
```

### i18n Audit

Verify no missing keys across all configured locales. Check all pages in every locale.

### Serve & Share

```bash
npm run dev &
npx cloudflared tunnel --url http://localhost:3000 &
```

Share in chat:
- Tunnel URL (for stakeholder click-through)
- Final bug summary (resolved + remaining with justification)
- Test results summary
- QA report

---

## 12. Common Pitfalls & Anti-Patterns

### Claude Code Pitfalls

| Pitfall | Prevention |
|---------|-----------|
| Edits memory/design instead of source code | Explicit prompt: "ONLY SOURCE CODE. Never touch memory, docs, or design files." |
| Claims bugs "already fixed" when not | Verify with file mtimes + browser re-test |
| `claude -p` drops harness context | ALWAYS tmux interactive mode |
| Bash permission dialogs stall session | `--dangerously-skip-permissions` flag at launch |
| Doesn't use OpenDesign MCP for design tasks | Post-Claude check for `mcp__open_design_*` calls |
| OAuth # trap (PKCE codes contain `#`) | Buffer method: `echo -n` → `load-buffer` → `paste-buffer`. Never `send-keys`. |

### Environment Pitfalls

| Pitfall | Fix |
|---------|-----|
| `.next/` cache stale → SWC vendor chunk errors | `rm -rf .next/ && npm run dev` |
| `next/image` blocks external image hosts | Add to `next.config.mjs` → `images.remotePatterns` |
| Cloudflare tunnel 530 after creation | Wait 5-10s, retry — tunnel needs time to establish |
| `cloudflared` binary not found | Use `npx cloudflared` |
| `/tmp/claude-state.json` stale from previous session | Delete before waiting for new completion |
| Next.js 15 `params` as Promise breaks route handlers | `await params` before destructuring |
| Next.js static caching returns empty dashboard | `export const dynamic = 'force-dynamic'` on API routes |

### QA Pitfalls

| Pitfall | Prevention |
|---------|-----------|
| Accessibility snapshot misses card grids | Use `browser_vision` for visual confirmation |
| Toast disappears before inspection | Check DOM `[role="status"]` directly |
| Locale-specific bugs missed | Test ALL locales (EN, FR, NL, etc.) |
| Fix verification uses cached page | Force full reload (`browser_navigate` with fresh page) |
| Console `const` collision from previous expression | Use unique variable names or `window.` prefix |
| DeepSeek doesn't support native vision | `browser_vision` falls back to auxiliary model; screenshot is always saved |

---

## 13. Tool Reference

| Tool | When to Use | Example |
|------|------------|---------|
| `clarify()` | User decisions needed | Vibe choice, Stack trade-offs, scope boundaries |
| `web_search` | Competitor research | "pokemon collection tracker site:github.com" |
| `delegate_task` | Parallel independent work | Research 3 competitors simultaneously |
| `browser_navigate` | Open app pages | `http://localhost:3000/en/dashboard` |
| `browser_click` | Test interactivity | Click "Add to Collection" button |
| `browser_snapshot` | Page structure inspection | Full accessibility tree with `full=true` |
| `browser_vision` | Visual inspection | Screenshot + analysis question |
| `browser_console` | JS errors + DOM inspection | Check fetch calls, console errors |
| `browser_type` | Form input testing | Type search query |
| `browser_scroll` | Scroll-based rendering | Lazy-loaded content, infinite scroll |
| `terminal` | Shell commands | dev server, git, curl, builds |
| `process` | Background processes | Dev server, Cloudflare tunnel |
| `memory` | Save durable facts | QA conventions, tool quirks, user preferences |

---

## 14. Communication Rules (Claude Dispatch)

### Buffer Method (mandatory for all Claude input)

```bash
# Write command to temp file
echo -n 'your command or code here' > /tmp/prompt.txt

# Load into tmux buffer
tmux load-buffer /tmp/prompt.txt

# Paste into Claude session
tmux paste-buffer -t claude

# Press Enter
sleep 0.3
tmux send-keys -t claude Enter
```

**Why:** `send-keys` corrupts PKCE codes containing `#`, and queues text in ways that break multi-line code. The buffer method is atomic and preserves all characters.

### Launch Template

```bash
# Check if session exists, create if not
tmux has-session -t claude 2>/dev/null || \
  tmux new-session -d -s claude -x 200 -y 40

# Launch Claude with bypass permissions
echo -n "cd /opt/data/projects/{project} && HOME=/opt/data/home claude --dangerously-skip-permissions" > /tmp/launch.txt
tmux load-buffer /tmp/launch.txt && tmux paste-buffer -t claude
sleep 0.3 && tmux send-keys -t claude Enter

# Handle dialogs
sleep 6 && tmux send-keys -t claude Enter   # Trust folder
sleep 3 && tmux send-keys -t claude Down     # Permissions → Yes
sleep 0.3 && tmux send-keys -t claude Enter
```

### Monitoring Template

```bash
# Check state (non-polling)
cat /tmp/claude-state.json
# {"status":"working"|"idle"|"prompt","ts":<unix_ms>}

# Read output (sparingly)
tmux capture-pane -t claude -p -J -S -30
```

---

## 15. Hover Effect Standard

All hover effects across the project must use:

```css
.card:hover {
  transform: scale(1.02);
  border-color: var(--color-accent);
  transition: transform 0.2s ease, border-color 0.2s ease;
}
```

**Forbidden:**
- ❌ `box-shadow` on hover
- ❌ `mix-blend-mode: screen` or sheen/gloss overlays
- ❌ `filter: brightness()` or `drop-shadow()`
- ❌ Animated gradient backgrounds

---

## 16. Appendix: Complete QA Checklist Template

```markdown
# QA Checklist — {project-name}

## Page: {page-name} — Locale: {locale}

| # | Element | Trigger | Expected Result | Actual | Status |
|---|---------|---------|----------------|--------|--------|
| 1 | Search input | Type "query" | Results appear | ... | ✅/❌ |
| 2 | Filter dropdown | Select option | List filtered | ... | ✅/❌ |
| 3 | Card tile | Click | Navigate to detail | ... | ✅/❌ |
| 4 | Add button | Click | Form opens | ... | ✅/❌ |
| 5 | Submit form | Click save | Toast + item added | ... | ✅/❌ |
| ... | ... | ... | ... | ... | ... |

## Visual Integrity

| Check | Page | Result |
|-------|------|--------|
| No text truncation | All | ✅/❌ |
| No element overlap | All | ✅/❌ |
| z-index sanity | All | ✅/❌ |
| Responsive (mobile) | All | ✅/❌ |
| Hover effect standard | All | ✅/❌ |

## Console

| Page | Errors | Warnings |
|------|--------|----------|
| / | None | None |
| /search | None | None |
| ... | ... | ... |

## API Health

| Endpoint | Method | Status | Response |
|----------|--------|--------|----------|
| /api/dashboard | GET | 200 | {...} |
| /api/collection | GET | 200 | [...] |
| ... | ... | ... | ... |
```

---

## 17. Appendix: pokemon-tracker Case Study

### Stats
- 34 cards tracked, €794.81 spent → €3,162.60 value (+298%)
- 8 user stories, 93 tests, 3 locales (EN/FR/NL)
- 9 bugs found in QA (2 CRITICAL, 4 MEDIUM, 3 LOW)
- Total problems encountered: 30 (11 foundation, 5 implementation, 9 QA bugs, 5 tool/infra)

### Key Lessons
1. **Always clear `.next/` before QA** — stale SWC cache was the #1 bug source
2. **Buffer method is non-negotiable** — OAuth `#` trap cost hours
3. **No `claude -p` ever** — harness context lives in `.claude/`, print mode drops it
4. **Visual QA catches what unit tests miss** — accessibility, hover, truncation, z-index
5. **Cross-page audit rule** — one bug → sweep all pages sharing that component
6. **Source code only directive** — Claude will edit memory/docs unless explicitly told not to
