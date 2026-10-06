# reclaiming-disk-space

An [Agent Skill](https://agentskills.io) that audits disk usage on any machine — **macOS, Windows, Linux** — and safely reclaims space through a visual, validated, trash-first workflow.

Stop paying for CleanMyMac or running random `rm -rf` commands. Give your agent a procedure: scan everything read-only, show you a dashboard, classify findings by risk, and delete **nothing** until you approve it item by item.

## What it does

Six phases, no shortcuts:

1. **Environment detection** — OS, disk total/free, % free (< 15% flagged critical)
2. **Multi-pass read-only scan** — macro → caches → dev tooling (Docker, package managers, AI runtimes) → large files > 500 MB → duplicates → orphan installers → trash/snapshots → recency checks
3. **Visual BEFORE report** — `disk-audit-before.html`: a macOS-style storage bar with an explicit *free* segment, a treemap of the biggest consumers, and a risk-classified plan
4. **Item-by-item validation** — the agent must ask which numbers to clean. No answer, no deletion
5. **Execution, trash-first** — files go to Trash / Recycle Bin / `gio trash`, never straight `rm`; one batch at a time with recovered size reported
6. **AFTER report** — same dashboard, before/after bars side by side, planned vs actual per item

### The dashboard

```
┌─────────────────────────────────────────────────────┐
│ Macintosh HD        406 GB used of 460 — 20 GB free │
│ ██████████████░░░░░░░░░░░░░░▒▒▒ ← free is explicit  │
└─────────────────────────────────────────────────────┘
 Treemap biggest → smallest, colored by tier
 Plan:  SAFE (green) / REVIEW (orange) / NEVER (gray)
```

## The three risk tiers

| Tier | Meaning | Examples |
|------|---------|----------|
| **SAFE** | Regenerable, zero data loss | package caches, build output, Docker build cache, browser caches |
| **REVIEW** | Costly to rebuild or needs human context | AI models, SDKs, device simulators, an old tool's data you replaced with another tool |
| **NEVER** | Never proposed | personal data, databases, SSH/GPG keys, cloud-synced content, active projects, running app caches |

Sorting rule enforced by the skill: **when in doubt → REVIEW, never SAFE.**

## Install

Works with any Agent Skills-compatible agent (Claude Code, OpenCode, Codex, Cursor, Gemini CLI, Copilot, and many more).

```bash
# open skills CLI
npx skills add <owner>/reclaiming-disk-space

# or add-skill CLI
npx add-skill <owner>/reclaiming-disk-space -g -a claude-code
```

Or manually: copy the repository folder into your agent's skills directory (e.g. `~/.claude/skills/reclaiming-disk-space`).

## Use

Just ask your agent naturally:

```
My disk is almost full, help me free up space
Free up some space before I install X
Show me what's eating my disk
```

The agent detects your OS, scans, opens an HTML dashboard in your browser, and waits for you to pick what to clean (e.g. `all safe` or `1,3,7`). Nothing is deleted until you answer.

## Design decisions

- **No hardcoded `rm -rf`** (unlike most prompt-based cleaners) — every removal goes to trash first; emptying the trash is a separate, reviewed step
- **Deterministic where it matters** — scan commands are fixed per-OS recipes in `references/platform-targets.md`; the agent only exercises judgment on classification
- **The dashboard is a static HTML file**, not a local web server — zero attack surface, shareable, works offline
- **Usage recency check before REVIEW** — a 10 GB folder isn't junk just because it's big; the skill checks last-used dates and active services (e.g. `ollama list`) before flagging AI models
- **`du` on APFS is treated as estimates**, and the agent is warned that macOS Settings percentages count purgeable space — `df` is the source of truth

## Safety model

Never touched, even if you ask: databases (SQLite/Postgres), SSH/GPG keys and credentials, dotfiles, `node_modules` of active projects, active Docker containers/volumes (removing a named volume = data gone), caches of running apps, cloud-synced content (local cache is fine, synced content is not).

The skill refuses "just purge everything" prompts and routes them back through validation.

## Structure

```
reclaiming-disk-space/
├── SKILL.md                        # the 6-phase workflow (loaded on trigger)
├── references/
│   └── platform-targets.md         # per-OS scan targets + cleanup commands
└── assets/
    └── dashboard_template.html     # the visual dashboard template
```

Progressive disclosure: only the `SKILL.md` metadata loads at startup; the platform reference loads when the OS is detected.

## Method sources

This skill was synthesized and hardened from the best existing open implementations — each partial on its own:

- [mad0ps gist](https://gist.github.com/mad0ps/5938d39d72a5f851ac65abc286b299d9) — cross-platform 3-tier structure and per-OS command tables (used `rm -rf` directly: replaced with trash-first)
- [glebis/claude-skills `disk-cleanup`](https://github.com/glebis/claude-skills) — dry-run-by-default, risk gating, trash-never-rm, preflight path validation (Unix-only scripts: generalized as platform reference)
- [xiaofenggan01/disk-cleaner-skills](https://github.com/xiaofenggan01/disk-cleaner-skills) — HTML dashboard and Mole-inspired path validation (Chinese docs, local server: replaced with static file)
- [How-To Geek (May 2026)](https://www.howtogeek.com/claude-found-50-gb-of-junk-on-my-pc-in-5-minutesjunk-bleachbit-missed) — the original viral multi-pass prompt with "do not delete anything" repeated twice

## License

MIT
