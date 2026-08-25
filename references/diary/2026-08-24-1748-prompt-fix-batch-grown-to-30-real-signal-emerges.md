---
description: Validated the prompt_prefix fix on the two worst pre-fix offenders, then ran a full 20-instance re-run and grew to 30 total with a size/problem-spread expansion. At n=30 the result is a real, consistent signal for the first time in this pilot -- harnessified resolves 70% vs baseline's 57%, at 57% fewer tokens and 32% less wall-clock. Also: a recurring, still-unexplained external kill of long-running batch background tasks, confirmed NOT caused by system sleep. Harness docs (SUITES.md, architecture.md, archiving-swebench-results/SKILL.md) brought current with all of today's changes.
date: 2026-08-24 17:48 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — SUITES.md/architecture.md/SKILL.md doc updates from this entry not yet committed)
---

Continuation of `2026-08-24-1404-prompt-prefix-added-to-stop-environment-archaeology.md`, same day.

**Validation.** Re-ran the two worst pre-fix offenders (`pallets__flask-4045`,
`matplotlib__matplotlib-25433`) against the new prompt_prefix batch (`20260824T190447Z-cdef683f`)
before committing further. Confirmed via transcript inspection, not just token deltas: **zero**
environment/test-verification Bash calls in either variant of either instance (no venv, no `pip
install`, no `pytest`, no Python-version hunting — all present before the fix). `flask-4045`'s
harnessified cost went from +252% vs. baseline to -16% (now cheaper); `matplotlib-25433` went from
+79% to +21%.

**Full re-run.** Opened a dedicated new batch (`20260824T193834Z-5b852589`, user's explicit call —
"a fundamental change" deserves its own batch, not folding into the ad-hoc validation one) covering
the same 20 instances as the pre-fix batch. 19/20 completed cleanly in one background `batch`
invocation; `pallets__flask-4045` timed out (a one-off, re-ran clean alone). At n=20: 6/20 instances
(30%) flipped PASS/FAIL versus the pre-fix batch, split roughly evenly in both directions — the
signature of n=1-per-instance noise, not a directional effect. Token/wall-clock reduction was the
one clean signal at this size: `harnessified` mean tokens 55,945 -> 31,139 (-44%), wall-clock 231.6s
-> 113.3s (-51%).

**Grown to 30.** User asked for 10 more instances "from a range of repo and problem sizes" — unlike
the earlier expansions (always each repo's smallest-gold-patch instance), this round deliberately
varied `patch` length too: `pallets__flask-5063` (3,443 chars, the largest available) down to
`django__django-11049` (530 chars), spanning small repos (flask, requests) through huge ones
(django x2 more, sympy) to medium ones (astropy, pylint, matplotlib, scikit-learn, pytest). All 12
SWE-bench Lite repos are now represented in this batch, several with more than one instance.
Ran in 4 waves due to the same recurring background-task kill (see below) — 3+7+2+5+3 instance
groupings across 5 separate `batch`/`run` invocations, archiving completed jobs after each kill
before relaunching the remainder. All 30 now archived.

**At n=30, the result reads as real for the first time in this pilot**: `harnessified` resolves
70% (21/30) vs. `baseline`'s 57% (17/30) — a genuine 13-point gap, not noise-cancelling — while
costing 57% fewer tokens (33,480 vs. 77,790) and 32% less wall-clock (123.8s vs. 181.4s) on the
answering step. 4 instances resolved for `harnessified` and not `baseline` (`matplotlib-25433`,
`pylint-7228`, `pytest-6116`, `sympy-19254`); zero instances went the other way. That one-directional
asymmetry, combined with the consistent cost reduction, is a materially different, more trustworthy
picture than the earlier 20-instance snapshot's coin-flip-shaped noise.

**A separate, still-open mystery.** Across today's later batch runs, the background `traversal-compare
batch` process was killed externally (not by this session, not the CLI's own nonzero-exit-on-error
path) four separate times, each stopping mid-run with some jobs completed and others never started.
First suspected macOS Power Nap/"Maintenance Sleep" while on battery — confirmed via `pmset -g log`/
`-g batt` that the machine WAS cycling through brief sleep/dark-wake states on battery, and the user
plugged in after that finding. But a subsequent kill happened with `pmset -g log` showing NO sleep
event anywhere near the kill time, on AC power the whole time — ruling out sleep as at least a
complete explanation. Root cause not identified; worked around operationally each time (archive
whatever completed via `results add-run`, relaunch the remainder as a smaller `batch`/`run`) rather
than solved. Single-job `run` invocations never got killed this way, only multi-job `batch` ones —
worth remembering as a practical mitigation (prefer smaller batches or sequential `run`s) even
without knowing why.

**Harness updated to match.** `SUITES.md` rewritten: the `patch_judge` section now describes the
wide-context-diff fix and `log_parser` dispatch (both landed after the section was first written);
the stale "not yet done: run the 7 new instances" replaced with an accurate "Two batches exist, and
they are NOT comparable" section naming both batch IDs, what each means, and pointing at
`results show-batch` as the one source of truth for exact coverage rather than restating a list
that will just go stale again. `references/architecture.md`'s `swebench_eval.py`/`swebench_judge.py`
rows updated for `context_lines`/`read_failing_test_excerpt`/log-parser dispatch, none of which were
mentioned when those rows were first written. `archiving-swebench-results/SKILL.md` gained
"changed the answering prompt" as a fourth concrete real-example reason for starting a new batch,
grounded in today's actual `prompt_prefix` batch split rather than a hypothetical.

**Not yet done:** repeated runs (`--times N`) to get a real per-instance accuracy signal, not just
the aggregate-direction one; the background-task-kill root cause; and per-repo statistical
significance (n=2-4 per repo is still thin for e.g. the `flask` 0/3 or `django` 25% results to be
read as more than "this batch's specific instances happened to be hard").
