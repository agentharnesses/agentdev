---
description: Dropped add-currency, leaving add-return-reason-code as efficient-exploration's sole task, and removed with-harness's per-subsystem skills/ layer entirely after finding it leaked task-relevant judgment (not just navigation) — references/ alone now carries the whole with-harness advantage.
date: 2026-08-19 15:19 CDT
git:
  agentdev: eb362cd (dirty — this entry's own changes not yet committed)
  traversal-compare: e798199 (dirty — this entry's own changes not yet committed)
---

## What happened

Working session flagged two problems with `efficient-exploration`, on top of just wanting one
task instead of two: the `with-harness` fixture felt overly nested (`skills/<subsystem>/SKILL.md`
adding a whole directory layer beyond `HARNESS.md` → `references/`), and there was an asymmetry
between the two variants that felt like more than the intended one.

Reading the actual skill files confirmed the second complaint was real, not just a feeling.
`skills/returns/SKILL.md` said outright: *"Only use this policy for a reason code where returning
to the exact originating warehouse actually makes sense (e.g. the warehouse shipped the wrong
item)"* — and separately named the exact decoy paths to avoid. That's not routing, it's the
answer's reasoning handed to the agent. A real confound beyond "routing files exist," in the same
family as the two mistakes `references/methodology.md` already documents (differently-organized
variants; force-triggering the metaskill on one side only).

## The fix

- Dropped `add-currency` entirely (per `references/methodology.md`'s own note that it "doesn't
  show a strong efficiency signal on its own") — `add-return-reason-code` is now
  `efficient-exploration`'s only task.
- Deleted `with-harness/skills/` (the index plus all 8 subsystem `SKILL.md` files) and the empty
  `.leaf-detectors` alongside it. `references/REFERENCES.md` + `references/architecture.md`
  already documented the same cross-subsystem call chain (`returns` → `inventory`'s
  `restock_returned_item()` → `shipping`'s `get_origin_warehouse()` for the `origin_warehouse`
  policy) without the prescriptive hint or the decoy call-out — so nothing needed inventing, just
  deleting the skill layer and trimming `HARNESS.md`'s `## Skills` section.
- `without-harness/` needed zero fixture edits — it never had `skills/` or `references/` in the
  first place. `diff -r` now reports exactly two "Only in with-harness" lines: `HARNESS.md` and
  `references`.
- Updated `task.yaml`'s `necessary_files` for `with-harness` to the flattened chain (`HARNESS.md`
  → `references/REFERENCES.md` → `references/architecture.md` → the two real target files),
  updated suite/repo docs (`suites/efficient-exploration/SUITES.md`, `suites/SUITES.md`,
  `traversal-compare/README.md`) to reflect one task instead of two, and rewrote
  `tests/test_tasks.py`/`tests/test_grading.py`, which hard-depended on the real `add-currency`
  fixture on disk and would otherwise have broken outright rather than just gone stale.

## Where this leaves things

`efficient-exploration` is now one task, with `with-harness` reduced to exactly what was asked
for: `HARNESS.md`, routing files (`references/`), and the metaskill — nothing fixture-specific
beyond that. Not yet committed as of this entry, and not yet re-run end to end with a real
`claude -p` invocation (that's a cost/time-bearing step left for the user to trigger manually,
per `skills/defining-a-test/SKILL.md`'s guidance to verify a real `traversal-compare run` before
calling a task `status: complete` — it already carried that status before this change, but the
fixture content under it changed enough that a fresh live run is worth doing).
