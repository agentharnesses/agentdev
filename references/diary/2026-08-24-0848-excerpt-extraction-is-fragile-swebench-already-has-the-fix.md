---
description: Digging into django__django-15061's failure found a real patch defect the judge missed, traced to a hand-rolled pytest-only regex in _extract_failing_test_excerpts. User asked directly whether framework-specific text parsing is fragile and whether SWE-bench itself has something better -- yes, and yes. SWE-bench's own dataset rows already carry a per-instance `log_parser` field naming the exact right parser; not yet wired in.
date: 2026-08-24 08:48 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — everything from this session's work not yet committed)
---

## What was found

Dug into `django__django-15061` (both variants FAILed, `patch_judge` said `addresses_problem: True`
for both). Reading the real Docker `test_output.txt` directly (not just `report.json`'s summary)
showed a genuine, shared defect: the actual bug (`BoundWidget.id_for_label` ignoring an id
`ChoiceWidget.options` already set — a *per-subwidget* issue) was in a different method than what
both AI patches touched. Both deleted `MultiWidget.id_for_label` (the *whole-field* label method)
entirely, expecting the fallback to `Widget`'s base implementation to behave equivalently. It
didn't: the rendered output went from `<label for="id_field1">` (expected) to a bare `<label>` (no
`for` attribute at all) — not the "unchanged id" the AI patches assumed, an outright missing one.
All 3 `FAIL_TO_PASS` tests failed identically for both variants (0/3 passed each) — not the
network-flake pattern from `psf__requests-2317`, a real, consistent, shared mistake.

`patch_judge` got this wrong because it never saw the actual assertion diff: `failing_test_excerpt`
came back empty for this run. Root cause: `swebench_eval._extract_failing_test_excerpts`'s header
regex (`^_{5,}.+_{5,}$`, matching pytest's `_______________________ test_name _______________________`
delimiter) is pytest-specific. Django's test runner is unittest-based
(`FAIL: test_name (module.Class)` / `======` separators) — a completely different format the
extractor silently doesn't recognize, returning `""` rather than erroring, so the gap was invisible
until judged output looked wrong on inspection. `sympy` (pytest-based) worked fine; this bug is
specific to non-pytest SWE-bench repos, which is exactly the direction this pilot just expanded
into (django/matplotlib/scikit-learn all have non-trivial fractions of non-pytest history, though
matplotlib/scikit-learn/sympy in this dataset all happen to use pytest — django doesn't).

## The question, and the answer

User's question, verbatim in spirit: are we doing framework-specific parsing on raw test output,
and isn't that fragile — is there something in SWE-bench itself we should be using instead?

**Yes to both halves.** Checked the installed `swebench` package directly rather than guessing:

- `swebench.harness.log_parsers` is a real module with per-framework/per-repo-family parsers —
  `python.py` alone has `parse_log_pytest`, `parse_log_django`, `parse_log_sympy`,
  `parse_log_matplotlib`, `parse_log_scikit`, `parse_log_requests`, `parse_log_astropy`,
  `parse_log_flask`, `parse_log_xarray`, and more (plus separate modules for JS/Java/Go/Rust/etc.,
  since SWE-bench covers non-Python repos too in its wider datasets). Each takes raw log text and
  returns a `{test_case: status}` mapping — this is exactly the mechanism that already produces
  `report.json`'s `tests_status` FAIL_TO_PASS/PASS_TO_PASS lists correctly across every repo in
  this pilot, without us writing a single line of parsing logic for that part. **That part was
  never fragile** — it's SWE-bench's own, already framework-correct.
- Crucially, **each dataset row already names which parser applies**: confirmed live —
  `psf__requests-2317` carries `log_parser: parse_log_requests`, `django__django-15061`/`14534`
  carry `parse_log_django`, `sympy__sympy-17139`/`18057` carry `parse_log_sympy`,
  `matplotlib__matplotlib-23314` carries `parse_log_matplotlib`,
  `scikit-learn__scikit-learn-13439` carries `parse_log_scikit`. This field exists per-instance in
  the raw parquet row — `swebench_dataset.SweBenchInstance` just never captures it
  (`load_instance` reads `repo`/`base_commit`/`problem_statement`/`patch`/`test_patch`/
  `fail_to_pass`/`pass_to_pass`/`environment_setup_commit`/`version`/`image`, no `log_parser`).

**What SWE-bench does NOT give us**: the parsers themselves discard failure detail — they classify
PASS/FAIL/SKIP per test case and throw away the traceback/assertion text, because that's all
`report.json`'s `resolved` verdict needs. The *excerpt* (a human/LLM-readable snippet of *why* a
test failed, which `patch_judge` needs as evidence) is our own bolt-on feature, genuinely not
something SWE-bench provides ready-made.

## Recommendation (not yet implemented)

Don't reuse `swebench`'s parser functions directly for excerpt extraction (wrong tool — they
classify, they don't excerpt) — but DO reuse the thing they prove already exists: **route excerpt
extraction by each instance's own `log_parser` value**, instead of one hardcoded pytest-only regex
guessing blind at every repo's log format.

Concretely, next session:
1. Add `log_parser: str = ""` to `SweBenchInstance`, read from the dataset row in `load_instance`
   (same pattern as `image`'s own optional-column handling).
2. Thread it through to `swebench_eval.read_failing_test_excerpt` (alongside `work_run_id`/
   `instance_id`/`failing_tests`, which it already takes).
3. Replace the single `_extract_failing_test_excerpts` pytest-only implementation with a small
   dispatch table keyed by `log_parser` value — a pytest-family extractor (already written, covers
   `parse_log_pytest`/`parse_log_sympy`/`parse_log_matplotlib`/`parse_log_scikit`/
   `parse_log_requests`, since these likely share output formatting even though SWE-bench gives
   them separate parser functions — worth confirming rather than assuming) and a Django/unittest
   extractor (new: `FAIL:`/`ERROR: test_name (module.Class)` header, block ends at the next such
   header or a `======`/`----` separator line). Unrecognized `log_parser` values fall back to
   today's pytest-shaped attempt (best-effort, matches the existing "return '' if nothing matches"
   discipline) rather than raising.
4. Re-judge `django__django-15061` (and any other archived non-pytest run) once this lands, to see
   whether `patch_judge` correctly flags the real defect once it can actually see it.

This doesn't change `swebench_eval.py`'s own grading at all (that path was never broken) — purely a
`swebench_judge.py`-supporting-infrastructure fix, same "diagnostic annotation only" boundary the
rest of that subsystem already respects.
