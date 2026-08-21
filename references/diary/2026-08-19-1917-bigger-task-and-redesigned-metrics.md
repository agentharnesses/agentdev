---
description: Expanded add-return-reason-code from a 1-step, 1-subsystem task into a 3-step task spanning returns -> notifications -> analytics (resume_previous chaining), and redesigned Metrics on the user's request — dropped precision_ratio/recall_ratio in favor of coverage_percent (recall reframed, no precision-style penalty for exploration), tokens, wall_clock_seconds, and end_accuracy (per-variant fraction of grading checks passed, filled in by cli.py after grading runs). Verified live: the bigger task passed 13/13 checks first try, then a second live run genuinely failed one check for without-harness, confirming end_accuracy reports partial credit correctly (8/9) rather than a flat pass/fail.
date: 2026-08-19 19:17 CDT
git:
  agentdev: 44fe58c (dirty)
  ahar-visualizer: 91ad3b9
  traversal-compare: fd43d33 (dirty)
---

## Part 1: a bigger task

Asked to make "the test" (`efficient-exploration/add-return-reason-code`, the only complete task
in the suite) more complicated — more steps, more operations. The existing fixture already modeled
8 subsystems (payments, notifications, inventory, returns, shipping, catalog, users, analytics)
identically in both variants, differing only by routing files — so this needed zero fixture
changes, only `task.yaml`.

Expanded from 1 step to 3, chained via `resume_previous: true` from step 2 on (same sandbox/session
throughout):

1. (existing) Add the `DAMAGED_IN_TRANSIT` reason code — `services/returns/`.
2. (new) Add a customer notification template + `render_return_damaged_in_transit` function,
   following the one existing template/function's format exactly —
   `services/returns/` -> `services/notifications/`.
3. (new) Add an ops analytics report tracking it by carrier, following the other 15 existing
   report files' format and getting listed in their index — `services/analytics/`.

Extended `necessary_files` per variant and added 5 new deterministic grading checks (13 total, up
from 8). Caught one real design bug before it shipped: `without-harness`'s fixture has no
`SERVICES.md`/routing files *at all* (that's the entire point of the fixture pair — see the task's
own description) — an early draft of the step-3 grading required updating
`services/analytics/reports/SERVICES.md` for `without-harness` too, which would have graded against
a file that variant's fixture is deliberately missing. Fixed by making that specific check
with-harness-only.

Updated `tests/test_tasks.py` (necessary_files lists, step count) and `tests/test_grading.py` (new
`_apply_full_three_step_solution` helper simulating all three steps' correct completion, since the
two tests that assert `result.passed is True` need every check's target file to actually exist —
correctly skips the `SERVICES.md` update for a sandbox that has no such file, mirroring the
with-harness-only check).

Ran live twice: first run passed all 13 checks on both variants, first try (62s/71s wall,
resume_previous chaining across steps confirmed working). Screenshotted the dev-host and saw a
noticeably richer multi-branch orange activity trail across the tree — not just one lit leaf like
the single-step task, spanning all three subsystems. (One screencapture along the way grabbed the
*real* VS Code window instead of the disposable dev-host — window focus isn't guaranteed to stay on
the dev-host for the duration of a multi-minute run; retook it and got the right window. Didn't
attempt any window-targeting automation to fix this, per `skills/dev-preview/SKILL.md`'s explicit
warning against it.)

## Part 2: redesigned metrics

User feedback: recall isn't a good metric — "the whole point is that the agent looks at files it's
supposed to," and the four metrics that actually matter are coverage percent, tokens, wall clock
time, and end accuracy. Asked two clarifying questions before touching the schema (both answered
with the recommended option):

1. Drop `precision_ratio` entirely; keep `total_touches`/`distinct_files_touched` as supporting
   detail (not headline metrics) since `coverage_percent` is computed from them anyway and they're
   useful when a run's numbers look off.
2. `tokens` = `input_tokens + output_tokens + cache_creation_input_tokens` summed across every
   step's reported `usage` (`cache_read_input_tokens` excluded — cheap reuse of a prior turn's
   context, not new work this run). `end_accuracy` = this variant's fraction of grading checks
   passed (partial credit), not a flat boolean.

`Metrics` (`metrics.py`) is now: `total_touches`, `distinct_files_touched`,
`necessary_files_touched` (detail) + `coverage_percent`, `tokens`, `wall_clock_seconds`,
`end_accuracy`, `reported_seconds` (headline, `end_accuracy` last since it defaults to `None`).
`compute_metrics` gained a `token_usages` parameter; `elapsed_wall_clock_seconds`/
`elapsed_reported_ms` renamed to `wall_clock_seconds`/`reported_ms` to match.

`end_accuracy` can't be computed inside `compute_metrics` — grading runs once, after *every*
variant's steps have already completed, so per-variant pass-fraction isn't known yet at that point.
`cli.py` now folds it into each variant's already-built `metrics` dict right after
`grading.grade_task()` returns, filtering `grade_result.checks` by `variant`.

Updated `tests/test_metrics.py` (renamed/rewrote every `compute_metrics` call site, added a
dedicated token-summing test), `references/architecture.md`, and
`skills/viewing-results/SKILL.md`.

## Verification

Ran live twice more against the now-bigger 3-step task with `--no-viewer` (since this pass was
about the metrics numbers, not the visual overlay):

- First: passed 13/13 both variants. `end_accuracy` correctly `1.0`/`1.0`.
- Second: `without-harness` genuinely failed to create the analytics report in step 3 (1 of its 9
  checks) while everything else passed — a real, organic failure, not a synthetic test. Confirmed
  `end_accuracy` reported `0.889` (8/9) for that variant rather than collapsing to a flat fail,
  and `1.0` (10/10) for `with-harness`, which used roughly 2.4x fewer tokens (13,782 vs 33,572) to
  get everything right. Full test suite green throughout (44 tests).

## Where this leaves things

Everything is local-only, not committed: `task.yaml`, `tests/test_tasks.py`,
`tests/test_grading.py`, `metrics.py`, `cli.py`, `tests/test_metrics.py`,
`references/architecture.md`, `skills/viewing-results/SKILL.md` all have uncommitted changes in
`traversal-compare`. Next step is committing and pushing, same as the two rounds of fixes earlier
today.
