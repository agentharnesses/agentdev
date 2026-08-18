---
description: Initial feature plan for the ahar-vsvis VS Code extension — harness inventory/flattening/tree visualization, plus a real-time fade-away visualizer and visit log for live agent navigation.
date: 2026-08-18 11:01 CDT
git:
  toprope-agentdev: 9d76788
  ahar-vsvis: bf9cdf5
---

## Where this content belongs

First pass at this plan was mistakenly written as `PLAN.md` directly in the `ahar-vsvis`
repo. Corrected: planning/design content belongs in this meta-repo's diary, not scattered
across product repos — `ahar-vsvis` should hold code (and eventually its own README once
there's something to describe), not planning docs. `PLAN.md` was removed from `ahar-vsvis`
and its content moved here.

## Feature plan

Nothing below is implemented yet — this is the target shape to build toward, from an early
design conversation.

### Harness structure

- **Harness inventory.** List every harness and sub-harness in the open repo, along with
  each one's routing files and leaves — the same information the `agent-harnesses` skill's
  `summarize.py`/`disclose.py` scripts compute, surfaced as a UI panel instead of a CLI dump.
- **"Flatten Harnesses" toggle.** A way to view harnesses organized by their own logical
  structure (harness → sub-harness → leaf) rather than by raw folder nesting — collapsing
  the parts of the directory tree that aren't harness-relevant.
- **Full directory tree visualization.** A visual tree of the whole repo that distinguishes
  `HARNESS.md` files, routing files, and leaf directories from ordinary files/folders (icons
  or badges), so harness structure is visible at a glance without opening files.
- **Click-through.** Clicking a file anywhere in the harness UI reveals it in VS Code's file
  explorer and opens it (and possibly a rendered/browser view — worth clarifying exactly
  what "in the browser" means once this gets built: an external browser preview, or an
  in-editor rendered webview).

### Live agent-navigation visualization

- **Real-time visualizer.** As a Claude Code session (observed via the hooks +
  transcript-tailing mechanism spiked in
  `references/diary/2026-08-18-1007-vs-code-base-or-extension.md`) reads/writes/lists
  files, show that activity live against the directory tree, with a configurable fade-away
  so recently-touched files are emphasized and older ones fade — a visual trail of how the
  agent is moving through the tree, not just a static "currently open file" indicator.
- **Visit log.** Alongside the live visualization, a chronological list of every file
  visited, in order. Clicking an entry in the log shows that file's content, so the log
  doubles as a scrubbable history of the session's navigation.

### Open questions to resolve while building

- Exact meaning of "see them... in the browser" above — needs a concrete UI decision.
- How harness inventory/flatten-view interacts with the live visualizer — are they the same
  tree view with an overlay, or separate panels?
- Multi-root workspaces and multiple concurrent Claude Code sessions: how the visit log and
  live visualizer disambiguate between sessions if more than one is running against the same
  repo.
