---
description: Implemented the log_parser-dispatch fix from the prior entry -- SweBenchInstance now carries the dataset's own log_parser field, threaded through to a new Django/unittest excerpt extractor alongside the existing pytest one. Live-verified against the real django__django-15061 archived run -- the judge's verdict flips from wrong to correct once given real evidence.
date: 2026-08-24 09:50 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — swebench_dataset.py/swebench_eval.py/cli.py changes + tests not yet committed)
---

Implemented the plan from the prior entry
(`2026-08-24-0848-excerpt-extraction-is-fragile-swebench-already-has-the-fix.md`):

- `SweBenchInstance` gained `log_parser: str = ""`, read from the dataset row in `load_instance`
  (same optional-column pattern as `image`).
- `instance_to_question_set` carries it into `swebench_eval_info`.
- `swebench_eval.py`: renamed the existing extractor to `_extract_failing_test_excerpts_pytest`,
  added `_extract_failing_test_excerpts_django` (Django's `FAIL:`/`ERROR: test_name (module.Class)`
  header format, block ends at the next header or a `===` separator -- confirmed live against the
  real `django__django-15061` log). `read_failing_test_excerpt` gained a `log_parser` parameter and
  dispatches on `_DJANGO_LOG_PARSER = "parse_log_django"`, falling back to the pytest extractor for
  everything else (which is what every other currently-used `log_parser` value -- requests/sympy/
  matplotlib/scikit-learn -- has been confirmed or is assumed to actually need).
- `cli.py`'s `_run_variant_swebench_eval` passes `info.get("log_parser", "")` through.
- New tests: a real Django-shaped sample log (transcribed from the actual failure this bug
  surfaced), extractor tests for both the Django block-matching and the dispatch, plus a test
  proving the pytest extractor genuinely returns `''` against Django output (pins the exact blind
  spot that caused the original bug, so it can't quietly come back unnoticed). `load_instance`/
  `instance_to_question_set` tests updated for the new field, including its own empty-column
  fallback test (mirroring `image`'s). Full suite: 285 passed, 1 skipped.

**Live-verified against the real archived run**, not just unit tests: re-extracted
`django__django-15061`'s real `test_output.txt` with `log_parser="parse_log_django"` -- got back
the real 3-test excerpt (each with the actual `<label for="id_field1">` vs `<label>` diff) that
came back empty before. Re-ran `swebench_judge.judge_patch` against both variants' real patches
with this real excerpt:

- `baseline`: now `addresses_problem: False, failure_attribution: patch_defect` -- correctly flips
  from the original (wrong) `True`.
- `harnessified`: `addresses_problem` stays `True` (its own reasoning is still a bit muddled about
  *why*), but `failure_attribution` now correctly reads `patch_defect` too -- the more important
  signal (this failure isn't environmental) lands correctly even where the boolean summary doesn't
  fully catch up.

Both variants' `failure_attribution` is now accurate, which is the field a human/analyst actually
acts on. The archived batch record for this run still has the old (empty-excerpt, wrong-verdict)
`patch_judge` output baked in -- there's no `results` CLI command to update an archived run's own
diagnostic fields in place, and refreshing it would mean a full expensive re-run, not just a
recompute. Left as-is for now; a future re-run of this instance will pick up the fix automatically.

**Not yet verified**: whether `matplotlib`/`scikit-learn` (both `log_parser` values not yet
directly observed in a real log, only assumed pytest-shaped by omission from the Django special
case) actually need their own extractor too, or if the pytest fallback is genuinely correct for
them -- the remaining 3 instances from the earlier batch (`matplotlib-23314`, `matplotlib-25433`,
`scikit-learn-13439`) haven't been run yet (batch process was killed externally mid-run, not by
this session), which will be the first real test of that assumption.
