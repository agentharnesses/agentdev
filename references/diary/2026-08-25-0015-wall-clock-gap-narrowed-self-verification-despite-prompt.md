---
description: Investigated why the harnessified/baseline wall-clock gap narrowed sharply on the 20 new instances added to 20260824T193834Z-5b852589 -- traced it to agents (harnessified ~2x as often as baseline) ignoring the "no working environment" prompt_prefix and successfully self-verifying via real test-suite runs, since several of the new Django instances actually have a working runner. Strengthened the prompt from a falsifiable environment claim to a flat "don't run checks" instruction plus an explicit finish-fast objective, opened a new batch seeded with the 6 instances that showed the problem, and stopped the old batch's remaining background jobs.
date: 2026-08-25 00:15 CDT
git:
  agentdev: b44ae87 (dirty -- this entry not yet committed)
  traversal-compare: e57ba0d (dirty -- dataset.yaml prompt_prefix change, batch 20260825T051502Z-af3ba59f creation, 20260824T193834Z-5b852589's 15 new archived runs, not yet committed)
---

Continuation of the `20260824T193834Z-5b852589` batch work from `2026-08-24-1748-prompt-fix-batch-
grown-to-30-real-signal-emerges.md`. User asked to grow that batch by 20 more random,
previously-unused SWE-bench_Lite instances and run them.

**Growing the batch.** Sampled 20 unused instances at random (270 candidates outside the existing
30), hit one real snag along the way: the sampling script's own diagnostic print lines (`used
count: 30`, `unused count: 270`) got captured into the same file the instance-id list was built
from, and `results add-instances` dutifully wrote both bogus strings into the batch's
`dataset.yaml` as if they were real instance ids. No `remove-instances` CLI command exists (only
`add-instances`), so fixed it the same way the batch's own `timeout_seconds`/`model` fields have
been hand-edited before -- a deliberate, documented repair, not routine editing. Worth remembering:
filter any list built from mixed stdout before handing it to a CLI that trusts its input positionally.

**Launched a 20-instance batch run.** First launch attempt silently collapsed all 20
`swe-bench-pilot/<id>` refs into a *single* argument -- not a `traversal-compare` bug, a zsh one:
the interactive shell here is zsh, which does not word-split an unquoted `$VAR` expansion by
default (bash does). Fixed by wrapping the launch in `bash -c '...'` explicitly. Real lesson for
this environment: any command built from a shell variable containing multiple space-separated
tokens needs an explicit bash subshell, not bare zsh.

**Monitored to completion via a self-paced `ScheduleWakeup` loop**, checking in every ~10-20
minutes, archiving each newly-completed run (`results add-run`) as it landed. No sign of the
previously-documented unexplained background-process kill this time; the run stayed alive for the
full ~3 hours needed. User asked to stop early once 15/20 were in and start a new, longer batch
session instead -- did so, killed the old batch process (PID 73051) cleanly (confirmed no orphaned
Docker containers or `claude` processes left behind), leaving 5 instances undone: 4 never started
(`sympy__sympy-15345/18087/20639/24152`) and 1 that errored mid-run (`sympy__sympy-12236`, "timed
out waiting for turn 2", no `result.json` written, not archived).

**The real finding: wall-clock gap narrowed hard, and it's not noise.** At n=30 (the prior entry),
harnessified beat baseline on wall-clock by -32% (123.8s vs 181.4s). At n=45 (with 15 of the 20 new
instances in), that gap fell to -10% (133.9s vs 149.4s). Split the new 15 out from the original 30
to isolate the driver: on the original 30, harnessified was still -32% faster; on the new 15 alone,
harnessified was **+80% slower** than baseline (mean), and slower on 12 of 15 individual instances
(41-48% slower even excluding the single worst outlier). This is a real reversal on the new
instances specifically, not a smoothing-out of noise.

Traced it to actual transcript gaps rather than guessing "resource contention" (the same discipline
named in `2026-08-24-0726`'s "pattern worth naming"). `django__django-16379`: harnessified took
99.1s vs baseline's 18.9s; a single 72.4s transcript gap traces to `python3 tests/runtests.py
cache.tests --parallel=1 -v1` -- it ran for real and returned real output.
`django__django-12708`: harnessified took 830.0s vs baseline's 231.2s; a single 268.6s gap traces
to the agent building its own venv (`$SCRATCH/djtest/bin/python`) and running Django's **entire**
test suite, no filter, `timeout: 600000`ms. Scanned all 9 new Django instances' transcripts for
self-verification-shaped Bash calls (`runtests.py`, `pip install`, venv creation): harnessified hit
6/9, baseline hit 3/9 -- roughly double the rate.

**Why this batch and not the last one.** The existing `prompt_prefix` (added 2026-08-24, see
`2026-08-24-1404`) told both variants "this sandbox... no working test runner... that environment
doesn't exist here." That's a factual claim about the sandbox, not an unconditional instruction --
and it was false for several of these new Django instances, which do have a real, working
environment reachable from the sandbox. The instances the prior fix was validated against
(`pallets__flask-4045`, `matplotlib__matplotlib-25433`) apparently didn't have one, so the same
self-verification attempt failed fast there and cost little. Here it succeeded and cost a lot.
Token cost stayed low regardless (self-verification output is cheap in tokens, expensive only in
wall-clock), which is why the token-savings story held up fine while the wall-clock story didn't.

**Fix applied, user-directed.** Reworded `suites/swe-bench-pilot/dataset.yaml`'s `prompt_prefix`
(both variants, still symmetric) from a claim about the environment to a flat, unconditional
instruction: don't run checks/tests/linters/programmatic evaluation at all, regardless of whether
the sandbox happens to have a working runner, "your work will be checked for you" -- plus an
explicit finish-fast objective, user's own wording: "Your objective is to come up with a complete
answer and finish as soon as possible." Opened a new batch, `20260825T051502Z-af3ba59f`, seeded
(per user's direction, "starting with some of the examples that were problematic in this batch")
with exactly the 6 instances that showed self-verification in `20260824T193834Z-5b852589`:
`django__django-12708/13220/13321/13757/14382/16379` -- a direct, targeted retest of whether the
new prompt actually stops the behavior on the cases where the old one didn't.

**Not yet done:** running the new batch; the 5 still-unarchived instances from the old batch
(4 never run, 1 timed out) remain declared in `20260824T193834Z-5b852589`'s coverage but un-run,
available to pick up later if that batch itself needs finishing rather than superseding.
