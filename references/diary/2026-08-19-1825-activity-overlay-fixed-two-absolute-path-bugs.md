---
description: Fixed and confirmed live the missing-activity-overlay bug from the prior diary entry. Root cause was two independent absolute-path mismatches between ahar-visualizer's node-tree ids and the file paths recorded in Claude Code transcripts — a macOS /var -> /private/var symlink mismatch (traversal-compare side) and the model sometimes passing a relative file_path to Read/Edit/Write (ahar-visualizer side). Fixed both, recompiled, reran the same task live in a fresh disposable dev-host, and confirmed via debug logs and a screenshot that the orange activity glow now renders correctly in both panels.
date: 2026-08-19 18:25 CDT
git:
  agentdev: eb362cd (dirty)
  ahar-visualizer: 375c6f7 (dirty)
  traversal-compare: e798199 (dirty)
---

## What happened

Picked up from the prior entry (`2026-08-19-1558-visualization-missing-activity-overlay.md`), which
had confirmed structural coloring worked but found no activity overlay, and paused before reading
`treePanel.ts`'s highlight-rendering path or `transcriptWatcher.ts`'s tailing logic.

Read both files. The wiring itself (`TranscriptWatcher` -> `onEvent` -> `postMessage({type:'step'})`
-> client's `resolveToNodeId`) was all correct. The actual bug: node ids in the client are always
the tree's absolute `fsPath` values (built from `rootPath`), and `resolveToNodeId` matches a
touched file's path by walking it up one directory at a time looking for an exact string match
against those ids — so any mismatch between the two path *strings*, even one that's filesystem-
equivalent, silently fails to resolve. Found two independent sources of that mismatch:

1. **`/private/var` vs `/var` (traversal-compare side).** `sandbox.create_sandbox()`
   (`traversal-compare/src/traversal_compare/sandbox.py`) stored the sandbox root unresolved. On
   macOS, `/var` is a symlink to `/private/var`, so a `claude` subprocess run with that unresolved
   path as `cwd` recorded every touched file's *resolved* path (`/private/var/...`) in transcript
   tool_use blocks. But the unresolved string was what got passed to the visualizer as `rootPath`,
   so every node id in the tree was built from the unresolved string too. Confirmed directly against
   a real transcript file before touching anything. Fix: `Sandbox.root = sandbox_root.resolve()` —
   the same treatment `compute_transcript_path()` already applied to `cwd` for the identical reason
   (that function's own comment cites an earlier diary entry, 2026-08-18, for the exact quirk — it
   just hadn't been applied to the sandbox root itself).

2. **Relative `file_path` values (ahar-visualizer side).** Confirmed live, on the *first* rerun
   after fix #1: the `with-harness` panel still showed zero resolved touches, this time because the
   model had called `Read`/`Edit` with plain relative paths (`./services/returns/returns_service.py`)
   rather than absolute ones — apparently not deterministic across runs, since an earlier transcript
   snapshot had recorded absolute paths for the same kind of call. `extractFilePaths()` in
   `transcriptWatcher.ts` assumed `file_path` was always absolute and never resolved it. Fix: each
   transcript line already carries its own `cwd` field, so resolve `file_path` against that via
   `path.resolve(obj.cwd ?? '', block.input.file_path)` (a no-op when already absolute) before
   returning it.

## Verification

Followed `skills/dev-preview/SKILL.md`'s safe pattern throughout — checked this session's own
process ancestry first (it lives in the real installed VS Code window, not a dev-host, so launching/
killing disposable instances was safe) and never attempted GUI automation against the new windows.

- Compiled (`npm run compile`), launched a disposable Extension Development Host
  (`--user-data-dir=/tmp/ahar-visualizer-demo-1`), confirmed the dev-queue heartbeat went live, and
  ran `traversal-compare run efficient-exploration/add-return-reason-code --ephemeral` live.
  Monitored both panels' per-panel debug logs (`/tmp/ahar-visualizer-debug-<slug>.log`) in real time
  rather than only checking after the fact. `without-harness` resolved every touch correctly
  (fix #1 confirmed); `with-harness` still showed `UNRESOLVED` for all three touches, all relative
  paths — this is where bug #2 was found and fixed.
- Applied the `transcriptWatcher.ts` fix, recompiled, killed the stale dev-host, launched a fresh
  one (`demo-2`), and reran the same task (this time without `--ephemeral`, so the sandbox would
  still exist afterward for a clean screenshot). Both panels' logs now resolved every touch,
  including the with-harness ones that had been relative paths before.
- Screenshot after the second run shows the orange glow trail (highlighted edges + a bright
  outlined node) descending from each tree's root down to the actual touched leaf, in both panels,
  on top of the pre-existing structural harness coloring (red root, orange routing nodes) — the
  overlay this tool exists to provide.

## Where this leaves things

Both fixes are applied and confirmed live end-to-end; not yet committed. `ahar-visualizer`'s working
tree has the `transcriptWatcher.ts` change; `traversal-compare`'s has the `sandbox.py` change (plus
unrelated pre-existing dirty state from other in-progress work — see this repo's own `git status`).
Two leftover disposable dev-host profiles at `/tmp/ahar-visualizer-demo-1` (killed) and
`/tmp/ahar-visualizer-demo-2` (left running, per the skill's guidance, in case whoever picks this up
next wants to look at it directly) — harmless clutter, `rm -rf` whenever convenient. Next step is
just committing both fixes.
