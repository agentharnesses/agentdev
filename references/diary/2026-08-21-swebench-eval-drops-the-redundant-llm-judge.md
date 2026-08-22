---
description: User correction -- swe-bench-pilot already has real ground truth (swebench_eval), so the LLM-judge check was pure redundant cost. Removed it entirely (no judge call happens now, not just an ignored result), and fixed a real latent bug the removal exposed -- overall_passed would otherwise vacuously report PASS on zero checks.
date: 2026-08-21 19:38 CDT
git:
  agentdev: 2a407d3
---

Direct continuation of today's wiring work. After walking through what "LLM eval" was actually
doing in this suite (`_auto_metric`'s auto-generated rubric, graded by `grading.py`'s judge against
the real gold patch), user's read: "I don't think we need LLM judgement on this particular one. We
already have the ground truth. In previous test we were using LLM as a judge because we didn't have
another evaluation." Correct — `_auto_metric` predates `swebench_eval.py`; it existed to give this
suite *some* correctness signal before the real one existed, and never got removed once it did.

Removed properly, not just made unused: `swebench_dataset.instance_to_question_set` no longer
attaches any check to the step at all (`checks=[]`, the dataclass default), and `_auto_metric`
itself is deleted. This matters beyond tidiness — `grading._judge_variant` returns early on an
empty check list, before ever calling `claude -p` (already true, already tested via
`test_judge_variant_returns_nothing_for_a_step_with_no_checks`), so this isn't "grade it but ignore
the result," it's "don't spend the judge call at all." Zero extra cost for this suite now, not just
zero extra signal.

Removing it exposed a real, previously-latent bug: `grade_result.passed` is `all(r.passed for r in
checks)`, and Python's `all([])` is `True` — so with zero checks, `overall_passed` would have
silently reported PASS regardless of whether the real fix actually worked. Every other suite always
had real checks, so this never fired before. Fixed in `cli.py`'s `_execute_run`: `overall_passed`
now folds in `_swebench_eval_all_resolved` (every variant's `swebench_eval.resolved is True`)
whenever the question set is dataset-native SWE-bench and `swebench_eval` was actually attempted —
an inconclusive result (`resolved: None`, from a caught `SweBenchEvalError`) counts as not-passed,
same as a genuine `resolved: False`, never a silent pass. Every other suite's `overall_passed`
computation is untouched.

Live-verified: `traversal-compare run swe-bench-pilot --variant harnessified` now shows
`grade.checks: []`, `end_accuracy: None` (correctly reflecting "no LLM checks", not a bug), and
`overall_passed: True` genuinely derived from `swebench_eval.resolved: True` — not the vacuous
`all([])` the pre-fix code would have produced. Updated `SUITES.md`, `architecture.md`,
`reporting-results/SKILL.md`, and `viewing-results/SKILL.md` (added the `harnessify_cost`/
`swebench_eval`/`overall_passed`-fold-in fields to the `result.json` reference, which had never
been backfilled when those features first landed) to match. 217 tests pass, including new coverage
for `_swebench_eval_all_resolved`'s four cases (all resolved, one unresolved, one errored, one
missing entirely).
