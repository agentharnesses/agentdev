---
description: Finished monitoring the strengthened-prompt full re-run (survived a usage-limit interruption mid-batch), then ran proper paired statistical tests on old-prompt vs new-prompt results -- found only the token-savings claim survives, resolved-rate and wall-clock deltas this project has been reporting as real wins are not statistically significant at n=44. Researched SWE-bench's own canonical methodology and found self-verification (Test Execution) is a standard, expected agent phase, not overhead -- the real mismatch is this project's bare-checkout fixture, not the model's instinct to verify. Decided to give agents the real per-instance Docker eval image as their own working sandbox instead of suppressing verification via prompt engineering.
date: 2026-08-25 08:41 CDT
git:
  agentdev: b44ae87 (dirty -- this entry not yet committed)
  traversal-compare: e57ba0d (dirty -- batch 20260825T051502Z-af3ba59f's archived runs, scipy/statsmodels added to .venv, not yet committed)
---

Continuation of `2026-08-25-0015-wall-clock-gap-narrowed-self-verification-despite-prompt.md`, same
day.

**Finished the strengthened-prompt full re-run.** Grew `20260825T051502Z-af3ba59f`'s declared
coverage from the original 6-instance sanity check to all 50 instances the old batch
(`20260824T193834Z-5b852589`) covered, then ran the remaining 44 at `--max-parallel 4`. Hit a real
usage limit partway through -- confirmed by literal transcript text ("You've hit your session limit
· resets 6:50am (America/Chicago)" and, for sessions that got further before the cutoff, "Checkpoint
reached"). Caught it by classifying every completed run's transcript tail before archiving rather
than trusting the batch runner's PASS/FAIL verdict at face value -- 9 of 44 jobs were corrupted this
way (some 0-token instant failures, one -- `sympy__sympy-12236` -- got 27 real turns and 582s of
work before being cut mid-fix). None of the 9 corrupted runs were archived; all were re-run after
the reset. Two instances (`sphinx-doc__sphinx-8721`, `sympy__sympy-18087`) then hit real timeouts
(unrelated to the usage limit) on top of that -- `sympy-18087` succeeded on its 3rd attempt,
`sphinx-8721` never did after 3 attempts and was left out of the batch (49/50 archived, one instance
permanently excluded for now).

**Statistical correction -- the headline wins from this whole investigation don't hold up.** User
asked for actual significance testing rather than eyeballing percentage deltas. Installed
scipy/statsmodels into the project venv (not yet added to `pyproject.toml`) and ran paired tests on
the 44 instances both batches actually completed:

- McNemar's exact test on resolved rate, old-prompt vs new-prompt, per variant: baseline
  64%→73% (p=0.289), harnessified 73%→64% (p=0.219) -- **neither significant**.
- Wilcoxon signed-rank on tokens/wall-clock, old vs new, per variant: token savings direction
  stable but the shift itself isn't tested here; **wall-clock old-vs-new not significant either**
  (baseline p=0.081, harnessified p=0.278).
- The more informative framing, run within each batch (baseline vs harnessified, paired by
  instance): **token savings is the only effect that survives** -- harnessified using ~50% fewer
  tokens than baseline is real and enormous (p<0.000001 in BOTH prompt conditions). Resolved rate
  (baseline vs harnessified, McNemar) is **not significant in either batch** (p=0.289 both times --
  same table, transposed). Wall-clock (baseline vs harnessified, Wilcoxon) is **not significant in
  either batch** either (old: p=0.959, essentially zero signal despite the -10.7% headline number
  this project has been reporting as a real win since `2026-08-21-swe-bench-pilot-a-real-5x-token-
  6x-time-win.md`; new: p=0.462).

This is a real correction to how this project's own numbers should be read going forward: at n=44
single-sample-per-instance, the token-cost advantage is the one finding that's actually solid.
Every resolved-rate and wall-clock claim in every prior diary entry about this pilot -- including
the oft-cited "70% vs 57%" and "-32% wall-clock" figures -- was read directionally from raw
percentages, never tested for significance, and turns out not to clear a proper paired test at this
sample size. Doesn't mean those effects are false, just that this project doesn't have the evidence
to claim them yet. Real fix: repeated runs (`--times N`) per instance, not more distinct instances --
a repeat gives each instance+variant+condition cell actual variance to test against instead of one
sample standing in for the whole distribution.

**Researched what SWE-bench itself recommends, since this whole investigation was chasing a
self-verification "problem" this project invented for itself.** SWE-bench's own docs are agnostic
about the agent's working environment -- the benchmark only grades the final patch, separately, in
a clean Docker container (exactly what `swebench_eval.py` already does correctly). But the
*canonical agent methodology* used by every major published baseline (SWE-agent, OpenHands,
Agentless) explicitly names **Test Execution as one of three standard phases**: Localization → Edit
→ Test Execution (reproduction + regression tests) -- not overhead to eliminate, a normal and
expected part of solving the task well. SWE-agent's reference scaffold runs the model inside a
real, pre-installed working Docker environment (custom Dockerfiles install actual project
dependencies plus tools like flake8) specifically so verification during solving works.

**The actual mismatch, reframed.** This project's fixture (`external_fixture.py`/`harnessify.py`)
gives agents a bare source checkout with nothing installed -- a deliberate simplification, not the
canonical setup. The self-verification "waste" chased across `2026-08-24-1404`,
`2026-08-24-1748`, and this morning's entry wasn't the model doing something wrong; it was the model
doing the standard, expected thing (Test Execution) against an environment that usually can't
support it. When the environment happened to actually work (several Django instances in the
20-instance expansion), self-verification wasn't wasted at all -- it was normal agent behavior in a
sandbox that, for once, matched the canonical assumption. Suppressing it via `prompt_prefix` (first
"claim it's broken," then "flatly forbid it regardless") was treating a symptom of the fixture
design, not the actual cause.

**Decision, user's call (offered the lighter option too -- soften the prompt instead of touching
fixture code -- user chose the architectural fix):** give agents the real, per-instance pre-built
Docker eval image as their own working sandbox, not just as `swebench_eval.py`'s separate grading
target. `swebench_dataset.py` already resolves this image per instance (`SweBenchInstance.image`,
threaded through to `QuestionSet.swebench_eval_info` for the grading step) -- the data is already
there, just not used for the agent's own execution environment yet. This is a real architecture
change (how the agent's session reaches the sandbox, not just fixture content), scoped separately
following this entry.

**Not yet done:** the Docker-sandbox implementation itself; repeated-runs (`--times N`) methodology
to get real significance on resolved-rate/wall-clock claims; `sphinx-doc__sphinx-8721`'s 3rd
consecutive timeout remains unexplained and unresolved.
