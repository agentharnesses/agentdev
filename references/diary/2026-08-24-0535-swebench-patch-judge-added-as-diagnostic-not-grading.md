---
description: Ran swe-bench-pilot's first real instance (psf__requests-2317); both variants failed execution grading but for different reasons -- baseline's patch was genuinely incomplete, harnessified's was correct but hit an unrelated httpbin.org network flake. Built swebench_judge.py, a diagnostic-only single-item LLM-judge sanity check, to tell the two apart -- never a grading signal, never affects overall_passed.
date: 2026-08-24 05:35 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — new module/tests/docs from this entry not yet committed)
---

`traversal-compare run swe-bench-pilot/psf__requests-2317 --variant both --no-viewer` (the first
real instance in `dataset.yaml`'s allow-list) came back `overall_passed: False` for both variants.
Digging into the real `swebench_eval` Docker logs (`test_output.txt`, `patch.diff` under
`/tmp/traversal-compare-swebench-eval/<run_id>-<variant>/logs/run_evaluation/...`, not just
`result.json`'s own summarized `report`) found two different root causes:

- `baseline`'s patch only fixed `PreparedRequest.prepare_method` in `models.py` (decode bytes then
  `.upper()`), but `Session.request()` in `sessions.py` still called `builtin_str(method)` *before*
  that function ever ran — on Python 3, `builtin_str(b'GET')` returns the repr string `"b'GET'"`,
  not `'GET'`, so a bytes method got mangled upstream of the fix and `test_encoded_methods` got back
  a 400. A real, genuine defect — baseline patched the wrong layer.
- `harnessified`'s patch fixed both spots correctly, and `test_encoded_methods` genuinely passed in
  its own run. Its one failure, `test_basicauth_with_netrc`, showed `assert 502 == 200` — httpbin.org
  itself returned a Bad Gateway during the test's third live HTTP call. Nothing to do with the patch.

User's question following this: how to handle this class of problem going forward, floating
re-introducing an LLM judge — but wanting it blind to harness condition, single-item (problem +
solution in, sanity-check out), not a comparison. Landed on a design distinct from both the
LLM-judge this suite dropped on 2026-08-21 (`swebench-eval-drops-the-redundant-llm-judge.md` — that
one was a *grading* signal, correctly removed as redundant with real ground truth) and `elo.py`'s
pairwise/Sort-MST machinery (built for cross-run *consistency*, wrong shape for single-item
correctness triage): a **diagnostic annotation**, run once per variant right after `swebench_eval`,
that never touches `overall_passed`/`resolved` either way.

Built `swebench_judge.py`: handed only the dataset's own `problem_statement` and the variant's own
patch diff, plus — only when unresolved — a bounded excerpt of the failing `FAIL_TO_PASS` tests' own
pytest output (new `swebench_eval.read_failing_test_excerpt`/`_extract_failing_test_excerpts`,
parsing pytest's own `___ test_name ___` FAILURES-section markers, capped per-test so a suite with
hundreds of failures can't blow up the prompt). Never told the variant id, never shown the chat
transcript, never told anything about harness/routing — blind by construction rather than needing
redaction, since a source diff against a pristine fixture has essentially nothing in it for a judge
to notice was harnessified, unlike the free-text-answer self-reveal gap `elo.py`'s own methodology
notes as still open for the other suites. Verdict shape: `{"addresses_problem": bool,
"failure_attribution": "patch_defect"|"environment_or_flake"|"inconclusive"|null, "reasoning": str,
"error": str|null}`. Same `claude -p --output-format json --setting-sources "" --strict-mcp-config
--disallowedTools ...` subprocess convention `grading.py`'s own `_judge_variant` already uses, same
fail-soft discipline (a failed/unparseable call comes back with `error` set, never raises).

Wired into `cli.py`: `_run_variant_swebench_eval` now also returns `model_patch` and
`failing_test_excerpt` (computed there, since it already has `result.report`'s `FAIL_TO_PASS`
failure list in scope); a new `_run_variant_patch_judge` runs right after the existing
`swebench_eval` `ThreadPoolExecutor` stage, storing each variant's verdict as `patch_judge` in
`result.json`. Skips the judge call entirely when `swebench_eval` didn't run or errored (Docker
down) — nothing coherent to sanity-check without a real execution verdict. `results show-run` now
prints `patch_judge` alongside `swebench_eval`, same "never dump the whole file" discipline (just
`addresses_problem`/`failure_attribution`/one-line `reasoning`, never the raw excerpt).

Updated the two existing `_run_variant_swebench_eval` tests for its now-larger return shape, added
new tests for `_run_variant_patch_judge` (both guard cases + the ordinary pass-through, mocking
`swebench_judge.judge_patch`) and a full `test_swebench_judge.py` (prompt shape, markdown-fence
stripping, invalid-attribution rejection, fail-closed on subprocess/parse errors — same mocking
discipline `test_grading.py` already uses, no real `claude -p` calls) and
`_extract_failing_test_excerpts`/`read_failing_test_excerpt` coverage in `test_swebench_eval.py`.
Full suite: 273 passed, 1 skipped. Docs updated: `architecture.md`'s module table (new
`swebench_judge.py` row), `suites/swe-bench-pilot/SUITES.md` (new section explicitly distinguishing
`patch_judge` as diagnostic-only from `swebench_eval`'s real grading, right under the paragraph that
explains why this suite dropped the earlier grading-signal LLM judge — so the two decisions read
side by side rather than looking contradictory).

Archived the original run into the existing results batch
(`20260823T213625Z-d139e5a9`) before any of this — `add-run` still only stores what `swebench_eval`
produced at the time, so that archived run predates `patch_judge` and won't have it; a fresh run
against the same instance would.

**Not yet done:** actually running `swebench_judge` live against a real judge call (everything above
is unit-tested with a mocked `subprocess.run`, matching this project's existing discipline for
judge-call code, but not yet exercised end-to-end via a real `traversal-compare run`). Also not yet
done: re-running `harnessified` alone to see whether the network flake clears on retry, and whether
`patch_judge` correctly flags `environment_or_flake` for it if not.
