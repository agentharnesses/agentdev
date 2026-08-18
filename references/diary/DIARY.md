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
