---
description: Renamed this meta-repo from toprope-agentdev to agentdev — the toprope branding
  had already been fully dropped from the product direction, so the repo's own name was the
  last stale leftover. Docs updated first (README, HARNESS.md, references/), then the GitHub
  repo itself renamed via gh repo rename.
date: 2026-08-18 23:45 CDT
git:
  toprope-agentdev: d3ef67e (dirty — this entry's own changes not yet committed)
---

## What changed

`HARNESS.md` and `README.md` had both explicitly called out for a while that this repo's own
name — `toprope-agentdev` — was a known-stale leftover of the dropped `toprope` product
direction (see `2026-08-18-1007-vs-code-base-or-extension.md`), deliberately left unrenamed
pending a decision. That decision was made today: rename to `agentdev`, dropping the dead
`toprope` prefix entirely, consistent with the same reasoning already applied to the
`ahar-vsvis` → `ahar-visualizer` rename (`2026-08-18-1629-...`).

Sequenced deliberately, docs before repo:

1. Updated the meta-repo's own current-state docs to drop `toprope-agentdev` in favor of
   `agentdev`: `README.md` (title + stale-name callout), `HARNESS.md` (frontmatter `name:` +
   the "known-stale... not yet renamed" paragraph, which also let the ahar-visualizer submodule
   line get corrected — it says "not yet added" but the submodule was in fact vendored in as of
   `d3ef67e`), `references/REFERENCES.md` (frontmatter description), and
   `references/ahar-visualizer-dev-workflow.md` (two literal path examples).
2. Renamed the GitHub repo itself: `gh repo rename agentdev --repo agentharnesses/toprope-agentdev`.
3. Updated the local `origin` remote to the new URL (`git remote set-url origin
   https://github.com/agentharnesses/agentdev.git`) so pushes go directly there instead of
   riding GitHub's rename redirect indefinitely.

**Deliberately not touched:** `references/diary/*.md` (historical record — same reasoning as
the `ahar-vsvis` rename entry: diary entries reflect what was true when written) and the
vendored submodules' own docs (`ahar-visualizer/README.md`, `vendor/metaskill/HARNESS.md`,
`vendor/cli/HARNESS.md`) that mention `toprope-agentdev` by name — those are separate GitHub
repos (`agentharnesses/ahar-visualizer`, `agentharnesses/metaskill`, `agentharnesses/cli`), out
of scope for a docs pass on this repo alone.

## Open follow-ups, not done here

- `ahar-visualizer/README.md`, `vendor/metaskill/HARNESS.md`, and `vendor/cli/HARNESS.md` still
  say `toprope-agentdev` — each lives in its own repo and needs its own edit + commit there.
- Whether the local checkout directory itself (currently still named `toprope-agentdev/` on
  disk) gets renamed to `agentdev/` is a separate, deliberately deferred decision — this
  session is running from inside that directory, so renaming it out from under the running
  Claude Code process (and whatever's holding the folder open, e.g. VS Code) carries the same
  category of risk flagged in `2026-08-18-1130-vscode-close-killed-session-mvp-recovered.md`.
  Not attempted here.
- The local checkout directory's name (`toprope-agentdev/`) is now out of sync with both the
  GitHub repo and its own docs — a reader `ls`-ing the parent folder will see the old name
  until that separate decision gets made.
