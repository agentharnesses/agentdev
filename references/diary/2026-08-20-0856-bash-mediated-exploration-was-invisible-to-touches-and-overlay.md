---
description: User's hunch, confirmed by digging through the transcript — Bash tool calls (grep/find/head) outnumbered Read calls in every variant of a real-repo-exploration run, and both metrics.py and ahar-visualizer's activity overlay only ever recognized Read/Edit/Write, making a real, asymmetric slice of exploration invisible to both. Fixed in both languages with a best-effort Bash-command file-path heuristic, verified against the real run that surfaced it.
date: 2026-08-20 08:56 CDT
git:
  agentdev: 3df61f6 (dirty — this entry's own changes not yet committed)
  ahar-visualizer: 91ad3b9 (dirty — transcriptWatcher.ts/test changes not yet committed)
  traversal-compare: 4603941 (dirty — metrics.py/test changes not yet committed)
---

## The hunch

User, after watching two live-visualized `real-repo-exploration` runs: "visability around
interaction is fairly sparse. I have a hunch that bash commands, rather than `Open`, are being
used." Asked to dig through the transcript for a consistent trend before doing anything else.

## Confirmed, with numbers

Both `metrics.py`'s `extract_file_paths` and `ahar-visualizer`'s
`TranscriptWatcher.extractFilePaths` (`src/transcriptWatcher.ts:267`) turned out to share the
identical restriction: `block.name in ("Read", "Edit", "Write")` — a `Bash` `tool_use` has no
`file_path` field, just an unstructured `command` string, so it was never recognized by either
system at all.

Counted real tool calls across the last live run (`20260820T132618Z-ddd447da`): `full-harness` 55
`Bash` / 32 `Read`, `metaskill-removed` 65 / 22, `harness-removed` 69 / 19. `Bash` outnumbers
`Read` in every variant, and the gap widens as routing support decreases. Dominant `Bash` verbs:
`grep` (136), `find` (71), `head` (49) — almost entirely exploratory reading, not shell
bookkeeping. Then a targeted check: real files that show up as a literal `grep`/`head` target but
*never* get a separate `Read` call. `full-harness`: zero. `metaskill-removed`: one (`SKILL.md`).
`harness-removed`: three (a diary entry, `reverse_disclose.py`, `summarize.py`).

**The asymmetry matters more than the raw gap.** The metaskill-guided condition explores less
broadly and loses almost nothing to this blind spot; the no-metaskill conditions explore more via
raw `Bash` and lose the most. Every `total_touches`/activity-overlay comparison reported earlier —
in this session's own reports, and presumably in every real-repo run before this fix — was
systematically *understating* exactly the conditions doing the most exploring. The true gap
between conditions was narrower than what was being shown, not wider.

## Fixed in both languages, deliberately incomplete

`metrics.py`'s `_extract_bash_paths` and `transcriptWatcher.ts`'s `extractBashPaths` — kept as
independent reimplementations, matching how these two files already duplicate rather than share
every other piece of extraction logic. A best-effort heuristic, not a shell parser (a `Bash`
argument has no structured path field to parse exactly): a candidate token must have a recognized
extension on its last path segment, must not contain a glob wildcard (`*`/`?` — so `find -name
'*.md'` is never mistaken for a real file), must not be a flag (leading `-`), and must not
immediately follow a bare `>`/`>>`/`<` redirect operator (an output file isn't something the
sandbox already had). A grep search-pattern argument never matches on its own, since it has no
extension.

Verified against the actual run that surfaced this, not just unit-tested: re-running
`compute_metrics` over the same real transcripts raised `total_touches` from 32→37
(`full-harness`), 22→31 (`metaskill-removed`), and 19→31 (`harness-removed`) — the two
no-metaskill conditions closed most of the gap, exactly the predicted asymmetry.

Documented explicitly as a heuristic lower bound, not an exact accounting — an unrecognized
extension, a path embedded in a larger quoted string (`python3 -c "..."`), or a command this
narrow ruleset doesn't match will still go uncounted. `total_touches`/the overlay are now a better
approximation than before, not a ground truth.

## Tests

`tests/test_metrics.py`: extraction from a grep target, ignoring search patterns, ignoring glob
wildcards, ignoring flag values, ignoring redirect targets, splitting across pipeline segments —
7 new tests, 87/87 passing overall. `ahar-visualizer/test/transcriptWatcher.test.js`: the same
cases end-to-end through the watcher (write a transcript line, tick, assert `filePaths`) — 3 new
tests, 111/111 passing overall (`npm test`, which compiles first).
