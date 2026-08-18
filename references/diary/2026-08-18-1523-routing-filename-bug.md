---
description: Found and fixed a real bug in routing-file naming — the agent-harnesses skill's own scripts (and the visualizer's hand-ported copy) named routing files after their immediate directory, not the top-level harness bucket. references/diary/DIARY.md renamed to REFERENCES.md; disclose.py/reverse_disclose.py/map_references.py and ahar-vsvis's harness.ts fixed; convention now documented in the metaskill.
date: 2026-08-18 15:23 CDT
git:
  toprope-agentdev: a013194 (dirty — this entry's own changes not yet committed)
  ahar-vsvis: 692b512 (dirty — harness.ts fix not yet committed)
---

## What was wrong

Reported by hand while looking at the `ahar-vsvis` tree visualizer: `references/diary/DIARY.md`
is misnamed. The real convention — confirmed against the `ahar` CLI (`agentharnesses.io`),
installed locally and treated as authoritative — is that a routing file's name is **not**
based on its own immediate directory. It's based on the top-level bucket directory (the one
sitting directly below the nearest harness-root boundary), propagated unchanged through every
nesting level beneath it. `ahar validate .` said so directly:

```
⚠ references/diary: grouping subdirectory is missing REFERENCES.md — add one to summarize its contents
```

`ahar show .` confirmed the positive case too: `skills/maintenance/` (not itself named
"skills") already correctly shows `SKILLS.md [routing]` — inherited from `skills/`, not
`MAINTENANCE.md`.

## It wasn't just the visualizer

Checked whether this skill's own Python scripts (`disclose.py`, `reverse_disclose.py`,
`map_references.py` — what `ahar-vsvis/src/harness.ts` was hand-ported from) already got this
right, since `skills/maintenance/SKILLS.md` was *already* correctly recognized as routing by
`summarize.py`'s output before any fix. Turned out that was a coincidence: `map_references.py`
had a hardcoded `or item["name"] == "SKILLS.md"` fallback that happened to paper over the bug
for the skills bucket specifically, while every other bucket (`references`, and — checked via
`ahar validate` — `ahar-vsvis`'s own subdirectories) stayed silently broken. All three scripts
actually computed the routing filename as `directory.name.upper() + ".md"` — the same
immediate-directory bug as the visualizer, just less visible because of that one hardcoded
escape hatch.

## What changed

- Added `find_bucket_name(directory, root)` to `disclose.py` (imported by the other two): walks
  up from `directory` to the nearest boundary (`root` itself, or an ancestor with its own
  `HARNESS.md`) and returns the name of whichever child sits on that path — the correct bucket
  name, resetting at nested harness-root boundaries. `should_skip`, `get_description`,
  `_get_context` (disclose.py), `spec_files_in` (reverse_disclose.py), and `build_tree`
  (map_references.py) all switched to use it; the hardcoded `"SKILLS.md"` fallback in
  map_references.py was removed as no longer needed (and actively wrong now that the real rule
  is implemented — it would let a stray `SKILLS.md` outside the skills bucket false-positive).
- `ahar-vsvis/src/harness.ts`: `buildHarnessIndex`'s recursive `scanDir` now threads a
  `bucketName` parameter down instead of using each directory's own `name`, resetting to a
  child's own name only when the parent is itself a harness root. Added `test/harness.test.js`
  (6 tests) — there was no dedicated test file for `harness.ts`'s classification logic before
  this, which is exactly how the bug went unnoticed.
- `references/diary/DIARY.md` renamed to `REFERENCES.md` (`git mv`, preserving history); the one
  reference to it by name, in `references/REFERENCES.md`, updated to match.
- Documented the convention in `.claude/skills/agent-harnesses/SKILL.md` under a new "Routing
  File Naming" section — it was completely unwritten before this, despite being load-bearing
  for every script in the skill. Also notes `ahar validate`/`ahar show` as the authoritative
  check when the real CLI is available, since this skill's scripts are a hand-rolled
  reimplementation for agents without it.

## Verification

`ahar validate .` no longer flags `references/diary`. `reverse_disclose.py` against
`skills/maintenance/modify-harness/SKILL.md` now correctly surfaces *both*
`skills/maintenance/SKILLS.md` and `skills/SKILLS.md` as routing ancestors — previously it would
have missed the closer one, since `spec_files_in("skills/maintenance")` was looking for a file
literally named `MAINTENANCE.md` that never existed. All of `summarize.py`, `disclose.py`
(start+select), and `reverse_disclose.py` re-run cleanly end-to-end post-fix. The `ahar-vsvis`
test suite (41 tests, all passing) includes the new harness.ts coverage plus the full existing
suite, unaffected.

Remaining, out of scope for this pass: `ahar validate` also flags `ahar-vsvis/{src,test,out}`
as missing `AHAR-VSVIS.md` summaries, and `ahar-vsvis/README.md` for a missing frontmatter
`description` — real findings, but not what was reported, left for a separate pass.
