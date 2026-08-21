---
description: Removed necessary_files and coverage_percent from traversal-compare entirely, on user judgment that the concept wasn't a meaningful signal — it's computed purely from the transcript (which files got touched) and is completely decoupled from end_accuracy (which is read from the sandbox's final state), so a low coverage number never meant anything was actually wrong, just that the model found a shortcut past files a human guessed would matter. Metrics is now three headline numbers: tokens, wall_clock_seconds, end_accuracy. Removed the field from task.yaml, tasks.py, metrics.py, cli.py, and every reference doc/skill; live-verified against the real task.
date: 2026-08-19 20:00 CDT
git:
  agentdev: 44fe58c (dirty)
  ahar-visualizer: 91ad3b9
  traversal-compare: fd43d33 (dirty)
---

## What happened

After presenting a thorough visual report of the 6-step run (as an Artifact, per user request —
`references/diary/2026-08-19-1943-symmetric-schema-and-six-steps.md`'s run), the user asked a
sharp question: how can a file be "necessary" and go unobserved while the run is still 100%
accurate? Answered inline at the time — `coverage_percent` and `end_accuracy` are computed by
completely independent code paths (one from the transcript, one from the sandbox's final state),
so there was never any logical connection between them; a model can shortcut past a file a human
guessed would matter and still get the right answer by inferring from a different, sufficient
source (e.g. copying `WRONG_ITEM_SHIPPED`'s exact field values from `reason_codes.yaml` without
needing to separately open `inventory_service.py` to understand *why* that policy works).

The user's follow-up: "I think the whole 'necessary document' idea isn't relevant. Remove it from
the yaml, and update harness resources accordingly." Not a request to fix or keep-as-secondary-
detail — a judgment that the metric doesn't earn its place at all, full removal.

## The removal

**Code:**
- `tasks.py`: `Task.necessary_files` field removed (it was already the only place the concept
  lived structurally, after the earlier per-variant → top-level restructuring).
- `metrics.py`: `Metrics.coverage_percent` and `Metrics.necessary_files_touched` removed;
  `compute_metrics()` lost its `necessary_files` parameter and the coverage-computation block.
  `Metrics` is now: `total_touches`, `distinct_files_touched` (supporting detail) + `tokens`,
  `wall_clock_seconds`, `end_accuracy` (the three headline metrics) + `reported_seconds`
  (secondary detail).
- `cli.py`: dropped the `task.necessary_files` argument from its `compute_metrics()` call; updated
  a stale comment referencing "four headline metrics."

**Data:** `task.yaml`'s top-level `necessary_files:` block (10 entries) removed from
`add-return-reason-code`; same from the `consistent-exploration` stub and the synthetic
`fake-suite` fixture in `test_resolve_task_bare_suite_name_raises_when_ambiguous`.

**Docs, "updated accordingly" throughout:** `references/methodology.md` (rewrote the "more
efficient agents" framing to name `tokens`/`wall_clock_seconds`/`end_accuracy` directly rather than
"how much of what it read was necessary," and trimmed the `necessary_files`-specific half of the
"declared once" bullet down to checks-only, with a short parenthetical explaining why the field is
gone rather than silently vanishing from the doc's history), `references/task-schema.md` (removed
the `necessary_files:` block from the annotated example), `references/architecture.md` (step 1 and
step 7's descriptions), `skills/defining-a-test/SKILL.md` (removed its now-obsolete
`necessary_files` guidance bullet entirely, keeping only the checks-symmetry one), `skills/
viewing-results/SKILL.md` and `skills/running-a-test/SKILL.md` (both `result.json` field
descriptions), and `suites/consistent-exploration/README.md`'s "what's left to author" list.

Each doc keeps a short explanation of *why* the field is gone (not just a silent deletion) — the
same reasoning each time: `coverage_percent` was independent of correctness by construction, since
`end_accuracy` is read from the sandbox's final state and never from what was touched to get there.

## Verification

Full test suite green (44 tests, same count as before — this pass only deleted assertions/
parameters, didn't add new behavior needing new tests). `tests/test_metrics.py` rewritten to drop
`necessary_files=`/`coverage_percent`/`necessary_files_touched` from every `compute_metrics()` call
and assertion; `tests/test_tasks.py` similarly. `validate` clean.

Given this was a pure removal (deleting a field and its computation, not adding new logic), skipped
a full 12-call two-variant live run and instead ran the real task with `--variant with-harness
--ephemeral` (6 calls) to confirm `result.json` still writes cleanly end to end. Confirmed:
`metrics` keys are now exactly `{total_touches, distinct_files_touched, tokens,
wall_clock_seconds, end_accuracy, reported_seconds}` — no `coverage_percent` or
`necessary_files_touched` anywhere — **PASS**, `end_accuracy: 1.0`.

## Where this leaves things

Everything in `traversal-compare` remains local and uncommitted — this is now four rounds of
changes stacked on the same working tree today (path-resolution bugfixes → metrics redesign →
schema symmetry + more steps → this removal). Genuinely due for a commit.
