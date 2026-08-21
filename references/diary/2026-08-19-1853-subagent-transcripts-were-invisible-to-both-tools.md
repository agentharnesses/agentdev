---
description: Found and fixed a third bug in the same family as the earlier activity-overlay fixes — a dispatched Task/Agent subagent's own Read/Edit/Write calls land in a separate <session-id>/subagents/<agent-id>.jsonl file, which neither ahar-visualizer's TranscriptWatcher nor traversal-compare's metrics.py ever read. This made with-harness recall look far worse than reality whenever the agent delegated exploration to a subagent (which the agent-harnesses metaskill often prompts it to do). Fixed both sides, verified against real captured transcript data (recall went from 1/7 to 5/7), committed and pushed both submodules.
date: 2026-08-19 18:53 CDT
git:
  agentdev: eb362cd (dirty)
  ahar-visualizer: 91ad3b9
  traversal-compare: fd43d33
---

## What happened

After committing and pushing the two path-resolution fixes from the prior session
(`2026-08-19-1825-activity-overlay-fixed-two-absolute-path-bugs.md`), the user asked to run the
efficient-exploration suite "for real." It passed, and the live overlay worked as intended — but
looking at the actual metrics in `result.json`, `with-harness`'s recall was a suspiciously low
0.14 (1/7 necessary files) against `without-harness`'s 0.5, despite `with-harness` getting the
right answer in fewer turns.

Traced it: the `with-harness` agent had dispatched an `Agent`/`Task` subagent tool call to do its
exploration ("Find return reason code implementation"). That subagent's own `Read` calls don't
land inline in the main session transcript at all — they're written to a completely separate file,
`<session-id>/subagents/<agent-id>.jsonl`, a sibling directory to the main `<session-id>.jsonl`
file (confirmed by inspecting the real transcript directory tree directly). Neither
`ahar-visualizer`'s `TranscriptWatcher` (flat, non-recursive `readdirSync` of the project dir) nor
`traversal-compare`'s `metrics.py` (reads only the snapshotted main transcript path) ever look
there. Pulling the subagent's transcript directly showed it actually read 12 files, including 3 of
the 4 `SERVICES.md`/routing files in the task's `necessary_files` list — the "1 touch, precision
1.0, recall 0.14" picture was an artifact of blindness, not the agent's real behavior. This isn't
harness-specific — any run that dispatches a subagent has this blind spot equally.

## The fix

**`ahar-visualizer/src/transcriptWatcher.ts`**: refactored the single-file tail loop inside
`tick()` (which mutated `this.offset`/`this.carry` directly) into a shared `tailInto(filePath,
state, filePaths)` helper operating on a `{offset, carry}` `TailState` struct, so the exact same
incremental-read/carry-partial-line logic works for any file. Added `findSubagentFiles(mainFile)`,
which derives `<mainFile's dir>/<mainFile's stem>/subagents/*.jsonl` and re-globs it every tick
(new subagent files can appear mid-run as the agent dispatches more of them). `tick()` now calls
`tailInto` once for the main file and once per discovered subagent file, accumulating into the same
`filePaths`/`step` for one batched `onEvent`. Session-switch handling clears `subagentStates` the
same way it already reset the main file's offset/carry.

**`traversal-compare/src/traversal_compare/results.py`**: `snapshot_transcript` now also copies the
live `<session-id>/subagents/*.jsonl` directory (if present) into a sibling
`<dest>/…-subagents/` folder alongside the main snapshot — same "live location isn't guaranteed to
persist" reasoning that already justified snapshotting the main file at all.

**`traversal-compare/src/traversal_compare/metrics.py`**: `compute_metrics` now also globs and
reads that `<transcript_path.stem>-subagents/*.jsonl` sibling directory for each transcript path,
feeding those lines through the same `extract_file_paths`.

Both `references/architecture.md` files and `ahar-visualizer/HARNESS.md` updated to document this.

## Verification

Ran the real `efficient-exploration/add-return-reason-code` task twice more with a fresh disposable
dev-host (same `skills/dev-preview/SKILL.md` pattern as before). Neither of these two runs happened
to dispatch a subagent (non-deterministic — the model sometimes explores directly, sometimes via
`Agent`), so they didn't exercise the new code path live. Rather than keep re-rolling the dice,
verified directly against the *real* subagent transcript already captured from an earlier run in
this session (`.../281f0855-42bc-44aa-82fe-7595c82e4d6d/subagents/agent-a686876915cb4d6bb.jsonl`):

- Python: called `results.snapshot_transcript` + `metrics.compute_metrics` directly against that
  real live transcript pair. `total_touches` went from 2 to 14; `recall_ratio` went from
  0.142857 (1/7) to 0.714286 (5/7).
- TypeScript: pointed a real `TranscriptWatcher` (via `os.homedir` override) at the same real
  session file and called `tick()`. Got the identical 14 touches / 12 distinct files as the Python
  side — full parity confirmed on real data, not just unit tests.

Full test suites green on both sides (108 ahar-visualizer, 43 traversal-compare, including 6 new
regression tests across both — subagent discovery, incremental tailing, session-switch state
clearing, and the results.py/metrics.py snapshot-and-read pipeline). Both committed and pushed:
`ahar-visualizer` `c1af24a..91ad3b9`, `traversal-compare` `5f11325..fd43d33`.

## Where this leaves things

Both fixes are live on `main` in their respective repos. The parent `toprope-agentdev` repo's
submodule pointers are now stale again (three commits behind across the two fix rounds) — same
open item as the prior diary entry, still not bumped. Also worth noting for whoever picks this up:
`without-harness`'s variant never dispatches subagents in this suite (no `agent-harnesses` metaskill
plugin loaded for it), so this blind spot was always harness-asymmetric in practice even though the
underlying bug wasn't harness-specific — worth keeping in mind when interpreting any historical
`with-harness` recall numbers from before this fix, since they're likely all understated to some
degree whenever that variant happened to delegate.
