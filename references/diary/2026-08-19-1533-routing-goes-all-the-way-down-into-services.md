---
description: Corrected a misreading of the previous entry's own fairness principle — "byte-identical besides routing files" applies throughout the whole with-harness tree, not just at the root, so services/ now gets a SERVICES.md at every grouping directory instead of staying deliberately unrouted.
date: 2026-08-19 15:33 CDT
git:
  agentdev: eb362cd (dirty — this entry's own changes not yet committed)
  traversal-compare: e798199 (dirty — this entry's own changes not yet committed)
---

## What happened

Supersedes `2026-08-19-1519-efficient-exploration-flattened-to-one-task.md`'s framing, not its
actual work (dropping `add-currency`, deleting the leaky `skills/` layer — that stands). What was
wrong was the follow-up reasoning about `services/`: after seeing `ahar validate` flag every
`services/*` subdirectory for a missing `SERVICES.md`, the working session's first instinct was
that this was *intentional* — that `services/` had to stay completely unrouted because it's the
shared, byte-identical content both variants navigate, and routing files anywhere inside it would
reopen the "differently-organized variants" confound `methodology.md` already documents as a
corrected mistake. That reasoning got written into `methodology.md` directly.

It was backwards. Asked directly, the actual rule is: everything stays byte-identical *besides
the routing files themselves*, and that applies throughout the whole harness, not just at the
root — routing files are the tested variable, wherever in the tree they live. `without-harness`
should still be missing every routing file, including ones nested arbitrarily deep in `services/`,
while every real content file (`.py`, `.yaml`, `README.md`, decoys) stays identical at every
depth. The earlier framing had quietly redefined "byte-identical" as "identical *and* routing
only exists at the top," which isn't what was asked and isn't what a fully standard-compliant
`with-harness/` would look like.

## The fix

Added a `SERVICES.md` at every grouping directory under `with-harness/services/` (13 files:
`services/` itself, each of the 8 subsystems, and 4 further-nested directories —
`analytics/reports/`, `notifications/templates/`, `payments/config/`, `returns/config/`) —
purely structural indexes (what's in this directory), deliberately not carrying the
cross-subsystem *reasoning* (that stays solely in `references/architecture.md`, to avoid
duplicating it inconsistently across multiple files, which a first pass at this did before being
caught and simplified). Also fixed a real bug this surfaced: `services/*/README.md` files still
pointed at `skills/*/SKILL.md`, deleted in the previous entry — now they point at the local
`SERVICES.md` or `references/architecture.md` as appropriate, identically on both variants.

`without-harness/` needed no new deletions to perform (nothing was ever written there), but the
mechanical-derivation recipe in `skills/defining-a-test/SKILL.md` and the fairness bullet in
`references/methodology.md` were both rewritten to state the general rule (delete every routing
file, wherever nested) instead of the old fixed list of four root-level names, which no longer
described what actually needs deleting.

Verified: `diff -r with-harness without-harness` now shows only routing-file paths (`HARNESS.md`,
`references`, and 13 `services/**/SERVICES.md` entries) with every real content file — including
all `services/*/README.md` — byte-identical; `ahar validate with-harness` reports zero "grouping
subdirectory is missing ...md" warnings (only pre-existing, unrelated "missing description in
frontmatter" style nits on ordinary content files); all 35 pytest tests pass; and a real
`disclose.py` session was walked by hand from `HARNESS.md` through `services/` → `returns/` →
`config/` to `reason_codes.yaml` to confirm the metaskill's progressive disclosure actually
reaches the real target file end to end, not just partway before falling back to prose.

## Where this leaves things

`efficient-exploration/add-return-reason-code` now has a `with-harness/` that's routed all the
way down, and a `without-harness/` that differs from it by nothing but the absence of every
routing file, at every depth. Not yet committed. Still not re-run end to end with a real
`claude -p` invocation — left as a manual step, same as the previous entry noted.
