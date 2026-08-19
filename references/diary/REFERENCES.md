---
description: Date/timestamped log of design thoughts, decisions, and plans for the toprope family of products — read chronologically to understand how and why the project evolved.
---

## Purpose

A running diary of the thinking behind toprope: fundamental ideas as they're first articulated, decisions made and why, plans drafted and revised. This is a record of *reasoning over time*, distinct from the rest of the harness (which records stable, current-state knowledge). When a diary entry's conclusion becomes settled fact, promote it into a proper reference or skill — the diary stays the historical trail, not the source of truth.

## Conventions

- One file per entry: `YYYY-MM-DD-HHMM-slug.md`, timestamp in the author's local time.
- Each entry starts with frontmatter:
  ```
  ---
  description: One-line summary of the entry's content, for disclosure listings.
  date: YYYY-MM-DD HH:MM TZ
  git:
    toprope-agentdev: <short-hash>   # this meta-repo, always included
    toprope: <short-hash>            # submodule, include if the entry concerns its source
    toprope-cli: <short-hash>        # submodule, include if the entry concerns its source
  ---
  ```
  Record the commit each relevant repo was actually at while writing the entry (`git rev-parse --short HEAD` in the repo; `git submodule status` from the meta-repo root for the submodules), not the commit the entry's own changes land in — the diary records what state was being worked on, and an entry is written before its own commit exists. Only list submodules the entry actually discusses; always include the meta-repo itself. If a repo has uncommitted changes at the time of writing, note that explicitly (e.g. `2759c54 (dirty)`).
- Entries are append-only history — don't edit past entries to reflect new thinking; write a new entry that supersedes or amends the old one, and note what it supersedes.
- Keep entries focused: one idea, decision, or planning pass per entry. Split unrelated thoughts into separate files even if written the same day.

## Index

- `2026-08-18-0939-vision-and-minimal-plan.md` — Founding vision (CLI-like editor + Claude Agent SDK + VS Code-style interface + agent-harness-native folder observability) and the minimal first plan.
- `2026-08-18-1007-vs-code-base-or-extension.md` — Resolved: toprope branding dropped; project becomes ahar-vsvis, a single VS Code extension demoing the agent-harnesses standard. Survey of open-source AI code editors, a verified hook+transcript-tailing spike (real `claude` CLI v2.1.234), and an honest risk layering of what's solid vs. provisional in that design.
- `2026-08-18-1101-ahar-vsvis-feature-plan.md` — Initial feature plan for ahar-vsvis: harness inventory/flatten-toggle/tree visualization, plus a real-time fade-away agent-navigation visualizer and clickable visit log. Also: planning docs belong here, not in product repos.
- `2026-08-18-1130-vscode-close-killed-session-mvp-recovered.md` — Incident: closing VS Code mid-session kills the Claude Code CLI session running inside it. The ahar-vsvis harness-tree MVP built in that session was recovered intact from the ephemeral scratchpad and committed (`ahar-vsvis` `fe8d5b9`). New harness rule: don't close VS Code while a session is running in it.
- `2026-08-18-1523-routing-filename-bug.md` — Found (via the real `ahar` CLI) that routing files must be named after the top-level bucket directory, not their own immediate directory — a real bug in this skill's own scripts, not just the visualizer. `DIARY.md` renamed to `REFERENCES.md`; `disclose.py`/`reverse_disclose.py`/`map_references.py` and `harness.ts` fixed; convention now documented in the metaskill's new "Routing File Naming" section.
- `2026-08-18-1629-ahar-vsvis-renamed-to-ahar-visualizer.md` — Renamed the extension (and its checkout directory here) from `ahar-vsvis` to `ahar-visualizer` — "vs" was redundant once the tool only runs inside VS Code. Local-only: directory, `package.json`, `aharVsvis.*` command/config IDs, and current docs renamed; the `agentharnesses/ahar-vsvis` GitHub remote deliberately left as-is for now, and past diary entries are left untouched as historical record.
- `2026-08-18-2345-toprope-agentdev-renamed-to-agentdev.md` — Renamed this meta-repo from `toprope-agentdev` to `agentdev`, dropping the last stale `toprope` leftover. Docs updated first (`README.md`, `HARNESS.md`, `references/`), then the GitHub repo renamed via `gh repo rename`. Local checkout directory and `origin` remote URL deliberately left unrenamed for now.
- `2026-08-19-1145-multi-panel-visualizer-and-traversal-compare.md` — Built `ahar-visualizer`'s side-by-side multi-panel capability (independently-tracked panels, transcript pinning, a URI handler for external scripting), then used it to bootstrap `traversal-compare` — a new submodule that sandboxes `claude` CLI sessions against paired with/without-harness fixtures to measure exploration efficiency and consistency. Two real bugs found by actually running it: a symlink path-normalization bug in the metrics, and a fixture missing the wiring to actually trigger the `agent-harnesses` skill.
- `2026-08-19-1230-dev-mode-queue-for-reaching-a-dev-host.md` — Confirmed `vscode://` URLs have the same "always routes to the main window" limitation already documented for `osascript`/Accessibility, so `traversal-compare`'s live viewer couldn't reach a disposable dev-host used to test `ahar-visualizer` itself. Fixed with a dev-mode-only file-queue IPC channel (`devQueue.ts` + a heartbeat-checked fallback in `viewer.py`), verified end to end: a real comparison run's two panels landed live in a dev-host, main window untouched.
