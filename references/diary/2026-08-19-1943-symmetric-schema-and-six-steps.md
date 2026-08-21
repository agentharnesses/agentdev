---
description: Restructured task.yaml's schema on user feedback — necessary_files and grading checks were declared per variant (with-harness's lists padded with routing files without-harness's fixture doesn't have), which the user correctly identified as wrong: both describe the task's outcome, a property of the task not of which variant produced it, so divergence (even "matching" divergence) is itself a confound. Moved both to single, variant-agnostic declarations (schema + code change, not just convention), and doubled add-return-reason-code from 3 to 6 steps spanning three more subsystems (payments, shipping, catalog). Verified live: 34 check evaluations (17 shared checks x 2 variants, perfectly symmetric), both variants end_accuracy 1.0.
date: 2026-08-19 19:43 CDT
git:
  agentdev: 44fe58c (dirty)
  ahar-visualizer: 91ad3b9
  traversal-compare: fd43d33 (dirty)
---

## What happened

Continuing from the metrics redesign (`2026-08-19-1917`), the user asked why `with-harness` and
`without-harness` had different check *counts* in a result. Answer at the time: by design —
`without-harness`'s fixture has no `SERVICES.md` at all, so a check requiring one variant to update
a routing-file index couldn't apply to both. The user pushed back on that being acceptable at all:
"the same exact tests should be done across both... these tests should be conducted in such a way
that they don't ask about harness enabled documents" — and separately, in the same turn as "double
the steps to 6," extended this to `necessary_files` too: "'this question should be looked at' and
'this value is required to be true' should be identical for both... updated in the design
constraints, and updated throughout."

This is a stronger claim than the "no routing-only checks" fix from two turns ago (which just made
the two variants' checks *match in shape* while still being declared twice). The real point: routing
files are the *mechanism* a variant may use to reach real content, never the necessary content or
the deliverable themselves — so declaring two lists per task, even nominally-identical ones, is
still wrong, because it invites silent drift and because `with-harness`'s `necessary_files` list
had in fact been padded with every routing file along its path (`HARNESS.md`, each `SERVICES.md`
hop) on top of the same real files `without-harness`'s list already had. That padding was silently
*deflating* `with-harness`'s `coverage_percent` for reaching the same answer *efficiently* —
touching fewer files, including fewer routing files — rather than rewarding it. This had been
present since the very first version of this task and was never caught until now.

## The fix

Restructured the schema itself rather than relying on authoring discipline to keep two lists in
sync:

- **`necessary_files`**: moved from each `variants[]` entry to a single top-level `task.yaml`
  field. `tasks.py`'s `Variant` dataclass lost the field; `Task` gained it.
- **`grading.deterministic.checks`**: each check no longer declares `variant` at all.
  `grading.deterministic_grade` now evaluates every check against every sandbox present in the
  run (naturally just one for `run --variant with-harness` alone), tagging each resulting
  `CheckResult.check` with which sandbox it ran against — a grading-time artifact for
  `cli.py`'s per-variant `end_accuracy` and for `result.json`'s reporting, not something a check
  declares up front.

Updated `references/methodology.md` (rewrote the "what makes a comparison fair" bullet — both
`necessary_files` and checks are now covered together, with the concrete before/after numbers as
the worked example), `references/task-schema.md` (the schema example itself), and
`skills/defining-a-test/SKILL.md` (steps 3 and 4's guidance). Also caught and fixed a stale
`skills/running-a-test/SKILL.md` reference to "precision/recall" that should have been updated in
the metrics-redesign pass two turns ago but was missed — "updated throughout" surfaced it.

Also fixed `suites/consistent-exploration/task.yaml` (the stub) to the same top-level-
`necessary_files` shape, and the synthetic `fake-suite` fixture in
`tests/test_tasks.py::test_resolve_task_bare_suite_name_raises_when_ambiguous`, for consistency —
neither is functionally load-bearing (both have empty `necessary_files`/`checks`) but the whole
point of this fix is that the schema shouldn't tolerate the old shape anywhere.

## Doubling the steps

Separately (same turn): "I want more steps. Double them, so 6." — extended
`add-return-reason-code` from 3 to 6 steps, all `resume_previous: true` from step 2 on, into three
subsystems it hadn't touched yet:

4. **payments** — `refund_category_for_reason_code(reason_code)`, returning `"carrier_fault"` for
   `DAMAGED_IN_TRANSIT`, so a carrier-caused refund is accounting-distinguishable from any other.
5. **shipping** — `flag_carrier_for_review(carrier_id)`, contract-only (docstring, stub body) —
   matches the existing `get_origin_warehouse`'s docstring-as-contract style in that file.
6. **catalog** — a `quality_issue_flag: false` field added to every product, plus
   `has_quality_issue_flag(sku)` in the catalog service.

`users`/loyalty was left untouched — no natural tie to a return-reason-code task without forcing
it (see `services/SERVICES.md`'s own framing: catalog/users are "consulted... independent of order
processing itself"), and six subsystems already gives good breadth. `necessary_files` grew to 10
real-content paths (all real files, no routing files, shared by both variants); grading checks grew
to 17 (also shared) — this task never needed the with-harness-only `SERVICES.md`-index check
from before at all once the schema was fixed, since checks don't carry `variant` anymore.

## Verification

Full test suite green throughout (44 tests — `tests/test_tasks.py` and `tests/test_grading.py`
updated for the new schema shape and the three new steps' solution simulation). Ran the real task
live end to end (`--no-viewer`, since this pass was about grading/metrics correctness, not the
overlay) — 12 sequential `claude -p` calls total (6 steps x 2 variants). **PASS**, `end_accuracy:
1.0` for both variants, 34 total check evaluations (17 shared checks x 2 variants — confirmed
exactly symmetric via `Counter`), zero failures. `coverage_percent` came back 80.0%/90.0% —
numbers that are now actually comparable to each other, unlike before.

## Where this leaves things

Everything in `traversal-compare` is local, uncommitted: `tasks.py`, `grading.py`, `cli.py`,
`task.yaml`, `tests/test_tasks.py`, `tests/test_grading.py`,
`suites/consistent-exploration/task.yaml`, and four reference/skill docs
(`methodology.md`, `task-schema.md`, `defining-a-test/SKILL.md`, `running-a-test/SKILL.md`). This
is now three rounds of changes stacked on the same uncommitted working tree today (bugfixes,
metrics redesign, schema symmetry + more steps) — worth committing as a coherent set rather than
letting it grow further uncommitted.
