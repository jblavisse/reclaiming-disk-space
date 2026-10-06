---
name: reclaiming-disk-space
description: Audits disk usage on any machine (macOS, Windows, Linux) and safely reclaims space in 6 phases — read-only multi-pass scan, visual HTML dashboard (macOS-style storage bar + treemap), SAFE/REVIEW/NEVER classification, item-by-item user validation, trash-first execution, before/after report. Use whenever the user mentions disk space, "disk almost full", "free up space", "clean up my disk", storage, reclaim space before an install, slow disk due to storage, or wants a visual before/after disk dashboard. Do not use for analyzing a single project folder (too narrow), migrating/archiving files to another volume, or photo library culling (use a dedicated triage skill for those).
---

# Reclaiming Disk Space

Cross-platform disk audit in 6 phases. No deletion without explicit item-by-item validation.
Freedom level: **medium** for scan and report, **low** for the execution phase.

## Inputs

- The target machine (local or remote via SSH)
- Optional requested scope (volume, folder, target quota)
- User context: any declared untouchable folders (cloud sync, vault, archives)

## Expected output

1. `disk-audit-before.html` — BEFORE dashboard (bar + treemap + plan)
2. Execution of validated items only
3. `disk-audit-after.html` — same dashboard with before/after bars side by side

## Procedure

### Phase 1 — Environment detection (read-only)

Detect the OS, then report: OS, total disk, free space, % free, RAM. Free < 15% = critical.

- macOS: `sw_vers | head -2` ; `df -h /System/Volumes/Data`
- Windows: `Get-PSDrive` ; `Get-CimInstance Win32_OperatingSystem | Select Caption`
- Linux: `cat /etc/os-release | head -2` ; `df -h /`

Pick scan targets from `references/platform-targets.md` based on detected OS.

### Phase 2 — Multi-pass scan (read-only)

Run all scans in parallel where possible, then do a second pass on the largest
directories found. Always:

- Macro pass: `du -sh ~/*` (or PS/Linux equivalent)
- Library/AppData/Caches/Containers pass
- Dev pass: package-manager caches, build, AI runtimes, Docker (`docker system df`)
- Files pass: files > 500 MB, duplicates (same name + size), orphan installers, logs
- Trash pass: ~/.Trash, Recycle Bin, local snapshots (`tmutil listlocalsnapshots /` or `vssadmin list shadows`)
- Usage pass: for each ambiguous large directory, check recency (`mdls -name kMDItemLastUsedDate`, `ls -lt`, history) before judging it

### Phase 3 — BEFORE visual report (nothing moves)

Generate `disk-audit-before.html` from `assets/dashboard_template.html` with:

- macOS-style storage bar: segments per category + explicit **free segment** (never merged into "system")
- Treemap of the biggest consumers, colored per tier
- 3-tier table with size, path, one-line reason:
  - **SAFE** — regenerable (caches, build output, installers)
  - **REVIEW** — costly to rebuild or context-dependent judgment (e.g. a folder of a tool replaced by another)
  - **NEVER** — personal data, keys, databases, cloud sync, active projects, system snapshots
- Totals: SAFE only, SAFE + REVIEW max scenario, projected % full after
- Conclusion first: recommended execution path in 1-2 sentences

Sorting rule: when in doubt → REVIEW, never SAFE.

### Phase 4 — Item-by-item validation

Ask which numbers to clean (e.g. `1,2,3` / `all safe` / `1-5,10`). NEVER execute without an explicit answer. A REVIEW item selected: confirm once more before acting.

### Phase 5 — Execution (imposed sequence)

Per validated item, in table order. `trash` rather than `rm` when possible
(macOS: `osascript -e 'tell app "Finder" to delete'` or `mv ~/.Trash`; Windows: Recycle Bin;
Linux: `gio trash`). Permanent deletion only upon explicit request.
Empty the trash separately, after review. One batch at a time, recovered size shown after each batch.

### Phase 6 — AFTER report

`df -h` + re-scan of touched categories, then `disk-audit-after.html`: same template,
before/after bars side by side, per-item recap (planned vs actual).

## Safety rules

- Never delete without explicit validation — even SAFE items go through phase 4
- Never touch: databases (SQLite/Postgres), SSH/GPG keys/credentials, dotfiles,
  `node_modules` in active projects, active Docker containers/volumes, caches of running apps
- Never go beyond the validated scope
- Docker: only `builder prune` + unused images; a removed named volume = data gone
- Cloud sync folders (pCloud/Dropbox/Drive): local cache deletable, synced content NEVER
- AI models (Ollama/HF/MLX): check `ollama list` and usage recency before REVIEW; active = NEVER
- Windows: warn if admin elevation is required before running

## Anti-patterns

- Asking the agent to "purge everything": refuse and go through phase 4
- Trust the macOS Settings %: it counts purgeable space; `df` is the source of truth for reality
- Treat `du` on APFS as exact sizes: they are estimates
- Invoke an interactive TUI (mole) from a script: it blocks on a pseudo-tty

## Method sources

- mad0ps gist "Disk cleanup skill cross-platform" — 3-tier structure + tables + per-OS commands
- glebis/claude-skills `disk-cleanup` — dry-run by default, risk gating, trash-never-rm, preflight validation
- xiaofenggan01/disk-cleaner-skills — HTML dashboard, colored bar, Mole-inspired path validation
- How-To Geek (May 2026) — original multi-pass prompt, "do not delete anything" instruction ×2
