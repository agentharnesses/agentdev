---
description: Built a third, real-repo comparison suite (this actual meta-repo and its submodules, exported programmatically from git history at a pinned commit, never committed as a static fixture) with three harness conditions instead of two, isolating routing-file presence from the metaskill's own progressive-disclosure mechanism — a live run caught a genuine wrong answer in the metaskill-removed condition, the first check failure in this project caused by an actually-wrong answer rather than an overly strict check.
date: 2026-08-20 07:58 CDT
git:
  agentdev: 4886940 (dirty — this entry's own changes, plus everything below, not yet committed)
  traversal-compare: 8b2bac1 (dirty — git_fixture.py, tasks.py/cli.py changes, new suite, doc updates, new tests, not yet committed)
---

## The user's critique of the earlier fixtures

Prompted by `2026-08-20-0057-wrong-turns-self-reveal-so-fixtures-need-silent-traps.md`: "I think
both these tests are too simple. They're not realistic." Proposed using `agentdev` itself and its
submodules, at a specific commit, as the fixture — with three sandboxes: full harness support,
metaskill removed, and metaskill + all agent-harnesses resources removed. Explicit constraint,
volunteered before I could even ask: "this should be defined programatically. I don't want a
literal coppy of this repo in this repo." Confirmed the pinned-commit requirement mattered
specifically because "one of the fundamental ideas of the test is to be repeatable."

## Design: three conditions, not two

Went through `EnterPlanMode` given the scope (a new fixture-source mechanism, not just a new task).
Two Explore agents confirmed the rest of the pipeline barely needed to change: `grading.py`,
`metrics.py`, `viewer.py`, and `cli.py`'s dispatch loop were already generic over an arbitrary
variant list; only two real hardcoded-to-two spots existed (`cli.py`'s `enable_metaskill = vid ==
"with-harness"` and the `--variant` `click.Choice`), plus two skill docs written assuming exactly
two columns/panels.

Landed on: two independent axes (routing files present/stripped × metaskill on/off), three of the
four combinations used (`full-harness`, `metaskill-removed`, `harness-removed` — the fourth,
metaskill-on-with-no-routing-files, is nonsensical). `full-harness` and `metaskill-removed` share
byte-identical fixture content; only runner behavior (`--plugin-dir` + priming) differs.

**New module `git_fixture.py`**: `git archive <pinned-sha> | tar -x` per repo, recursively for
every submodule (SHAs read via `git ls-tree <ref>` against the ref's own recorded gitlinks, not
today's working-tree checkout — two of `agentdev`'s five submodules had already drifted locally by
the time this was built, confirming why that distinction is load-bearing, not paranoia). Cached
outside the repo, content-addressed by the resolved SHAs. `tasks.py` gained `fixture_source` as an
alternative to `fixture_path` (resolved lazily, only when a run actually needs it) and a `metaskill:
bool` field on `Variant`, defaulting to the old `id == "with-harness"` rule so both existing tasks
needed zero edits.

Pinned commit chosen deliberately, not derived: `agentdev @ 4886940` (this session's own earlier
push) — verified reachable on `origin/main`, and every one of its five submodule pins independently
verified reachable on their own `origin/main` too, before committing to it as the task's fixed
artifact.

## Two real bugs, found only by testing against the actual repo — not visible from reading the code

1. `shutil.copytree`'s default `symlinks=False` silently **dereferenced** a real git-tracked
   symlink this repo actually has (`.claude/skills/agent-harnesses -> ../../vendor/metaskill/agent-harnesses`)
   into a duplicated real copy, instead of preserving the symlink the way the raw export actually
   has it. First noticed as an inexplicable "stripped has *more* files than raw" — traced to the
   copy step, not the deletion step. Fixed: `symlinks=True`.
2. A single-pass routing-file-deletion walk that deletes each `HARNESS.md` as it's encountered
   corrupts nested-boundary detection for every file visited afterward under that now-vanished
   boundary — a deeply-nested `SERVICES.md` (inside `traversal-compare`'s own fixture-of-fixtures
   test content, itself a nested harness) silently survived stripping because its enclosing
   `HARNESS.md` had already been deleted by the time its boundary was checked, so the walk searched
   straight past it to an outer ancestor and computed the wrong propagated name. Caught by directly
   diffing raw vs. stripped file lists, not by trusting a hand-traced read of the code (which
   looked correct and wasn't). Fixed: scan the whole tree for what to delete first, then delete in
   a separate second pass.

## Six real questions, verified against the actual pinned snapshot before locking in checks

Not manufactured — each traces something that actually happened in this project's own real history:
the routing-filename propagation rule (`find_top_level_dir_name` in `disclose.py`), a
`.harnessleaf`-vs-`.leaf-detectors` precedence scenario, which of four plausible candidates
`--plugin-dir` actually points at (three similarly-named `vendor/*` dirs are all wrong — it's
`traversal-compare`'s own local copy), the real diagnosed root cause of the `-p -r` slowness
regression (only in `methodology.md`'s narrative, not code), the `devQueue` fallback for a stale
`vscode://` handler (only in a diary entry), and harness-priming's exact prompt wording.

## Live result: 26/27 checks passed, and the one failure is real, not a check bug

Ran `real-repo-exploration/agentdev-snapshot --variant both` live, all three variants, `--ephemeral
--no-viewer`. `full-harness` and `harness-removed` both correctly named `find_top_level_dir_name` in
`disclose.py` for question 1. `metaskill-removed` instead confidently named `map_references.py`'s
`build_tree` — checked the actual source: `build_tree` *calls* `find_top_level_dir_name` to compute
`routing_name`, it doesn't implement the boundary-walk logic itself, so the answer was genuinely
wrong, not just differently phrased. The agent even claimed to have verified it by building and
running a throwaway test tree.

Unlike `platform-lookup`'s Q6 (where a check was too strict against an otherwise-correct answer,
and got fixed), this is the first check failure in the project caused by an actually-wrong answer —
so the check was left as-is rather than "fixed." Whether this generalizes (metaskill-removed being
more prone to this kind of plausible misattribution than either full-harness or harness-removed) or
was one noisy run is exactly the kind of question this suite exists to keep collecting data on —
treat as one data point, not a trend, same discipline applied throughout this project's prior
volatile timing results.

## Where this leaves things

`git_fixture.py`, the `tasks.py`/`cli.py` generalization, the new suite, all doc updates
(`task-schema.md`, `architecture.md`, `methodology.md`'s new "Real-repo, three-condition split"
section, `defining-a-test`/`running-a-test`/`reporting-results` skills), and new tests
(`test_git_fixture.py`, `test_tasks.py` additions) are all built, tested (80/80 passing), and
verified live end-to-end — not yet committed as of this entry.
