---
description: Live comparison run + dev-host visualization confirmed the structural harness-tree coloring (routing vs. non-routing nodes) renders correctly via devQueue, but no activity/navigation overlay (which files the agent actually touched) appeared on top of it — the actual point of the tool. Investigation paused mid-way, not yet fixed.
date: 2026-08-19 15:58 CDT
git:
  agentdev: eb362cd (dirty)
  ahar-visualizer: 375c6f7
  traversal-compare: e798199 (dirty)
---

## What happened

Ran `efficient-exploration/add-return-reason-code` for real (PASS both variants) with a fresh
disposable dev-host live. Confirmed via screenshot the structural coloring is correct — red
harness-root node, orange routing-directory nodes throughout `with-harness`'s tree, plain gray
throughout `without-harness`. But flagged (independently, by the user) that this doesn't look like
the visualization actually "worked" — there's no visible activity trail (the "visited, now cold" /
"currently hot" legend states) showing which files the agent's transcript actually touched during
the run, which is the tool's actual purpose per its own description ("real-time fade-away
agent-navigation visualizer").

Verified the underlying data is fine: both transcripts (`with-harness-step1.jsonl`,
`without-harness-step1.jsonl`) exist with full real content (65455 / 63817 bytes) at exactly the
live path (`~/.claude/projects/<slug>/<session-id>.jsonl`) that `viewer.py` computed and pinned
*before* the run started, via the same `compute_transcript_path` slug logic on both the Python and
TypeScript sides. Read `devQueue.ts` and `extension.ts`: confirmed the dev-queue request's
`sessionFile` is passed straight through to `HarnessTreePanel.createCustom({ rootPath, sessionFile,
label })`, same as the URI-handler path. So the plumbing up through panel creation looks correct —
the gap is somewhere inside `treePanel.ts` (1791 lines, not yet read) or `transcriptWatcher.ts`
(221 lines, not yet read): either the watcher isn't actually tailing/parsing the pinned file
correctly in this flow, or the render logic isn't picking up parsed touch events.

## Where this leaves things

Not fixed yet — paused for a usage-limit checkpoint right as the investigation was about to move
into `treePanel.ts`'s highlight-rendering path and `transcriptWatcher.ts`'s tailing logic. Next
step: read those two files specifically for how `sessionFile`-driven (vs. auto-follow-latest)
panels wire up transcript watching, since this dev-queue-driven multi-panel path is newer than the
originally-verified single-panel/URI path and may not have gotten the same real end-to-end check
for the activity overlay specifically (only for "does a panel open at all").
