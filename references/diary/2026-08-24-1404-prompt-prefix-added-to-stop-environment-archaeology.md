---
description: Dug into why harnessified cost more than baseline on several instances after the first 20-instance batch -- not sparse routing (checked flask's authored routing directly, it's genuinely detailed), but agents spending real Bash calls building venvs and hunting Python versions to self-verify in a sandbox that was never going to have a working environment. Added a symmetric prompt_prefix to both variants telling the agent not to bother, and opened a new batch for the methodology change.
date: 2026-08-24 14:04 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — dataset.yaml prompt_prefix/timeout_seconds change not yet committed)
---

User asked to back up the "harnessify routing gets sparse on bigger repos" theory from the prior
session with real evidence. Checked directly rather than defend the theory: `pallets/flask`'s
authored `HARNESS.md`/`src/flask/SRC.md` are genuinely detailed (per-module line counts and
responsibility summaries, not generic filler) — the theory was wrong, at least for this repo.

Read the actual transcripts instead. `pallets__flask-4045`: `baseline` made 9 Bash calls, read 3
files, edited 4, and stopped. `harnessified` made 35 Bash calls, including `which python3.9
python3.8 python3.10`, building a scratch venv, `pip install -q "werkzeug==2.0.3" ...`, and running
`pytest tests/test_blueprints.py` repeatedly against specific test functions. `matplotlib__
matplotlib-25433` showed the same shape (baseline 42 Bash calls, harnessified 69), though there both
variants were already doing heavy verification, so the gap was smaller proportionally.

The real driver: the sandbox during the *solving* step is a bare source checkout with nothing
installed — no venv, no dependencies. The actual verification (a correct, prebuilt Docker
environment, `instance.image`) only exists later, in `swebench_eval.py`'s separate grading step,
which the answering session never sees. An agent that tries to "verify" by running tests is paying
real cost to reconstruct an environment from scratch that already exists, correctly, one step later
— pure overhead relative to what's actually being measured (can the agent solve the bug).

Discussed three fixes, roughly cheap-to-expensive: (1) prompt guidance telling the agent not to
bother, using `Variant.prompt_prefix` — already a real mechanism, prepended to the first step's
prompt (`cli.py:178`), just never used by this suite; (2) run the agent inside the instance's own
real Docker image so verification is cheap and legitimate instead of archaeology — the "correct"
fix, matching how real coding-agent SWE-bench harnesses work, but a real architecture change
(`claude_runner.InteractiveSession` runs on the host, not in a container, today); (3) detect and
report environment-setup-shaped tool calls as a separate cost, same discipline `harnessify_cost`/
priming already get, without changing agent behavior. Went with (1) now — cheapest, most targeted,
and directly undoes the exact overhead just found live.

Added an identical `prompt_prefix` to BOTH `baseline` and `harnessified` in `suites/swe-bench-pilot/
dataset.yaml` (a fair, symmetric change, not something that advantages either condition) telling
the agent the sandbox has no working environment, not to build one, and that verification happens
separately later. Also bumped the suite-level template's own `timeout_seconds` 600 -> 1800 to match
the batch-level fix from earlier today (the template itself still had the old, too-tight default).
Live-verified the wiring end to end (`build_question_set_from_config` -> both variants carry the
prefix) — 285 tests still passed.

Since this changes what the agent is told, it's a real methodology change, not a tweak — per this
project's own batch discipline ("start a new batch rather than mixing incomparable results"), opened
a fresh batch (`20260824T190447Z-cdef683f`) declaring the same 20 instances as the prior one, rather
than mixing pre/post-prompt-change results into the same rollup. Prior batch (`20260823T213625Z-
d139e5a9`, 20/20 run, 70% pass rate) stays as the pre-fix baseline for comparison.

**Not yet done:** actually running anything against the new batch. Given the machine already burned
a full day's worth of compute on the first 20-instance pass, next step should probably be validating
the fix cheaply first (re-run just the two worst offenders, `pallets__flask-4045` and
`matplotlib__matplotlib-25433`, and confirm the Bash-call-count/token gap actually shrinks) before
committing to a full 20-instance re-run.
