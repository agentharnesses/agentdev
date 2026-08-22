---
description: Wired swebench_eval.py's execution-based grading into cli.py's real run/batch/elo path (previously a standalone, unwired module) and confirmed a single live `run` now produces LLM-judge grading, execution-based grading, and the full metrics set (tokens broken down, wall-clock, files touched) together in one result.json.
date: 2026-08-21 19:03 CDT
git:
  agentdev: 2a407d3
---

Direct continuation of today's earlier `swebench_eval.py` work
(`2026-08-21-swebench-eval-live-verified-both-directions-apple-silicon-fixed.md`), which proved the
module correct in isolation but explicitly left it unwired — sandboxes are ephemeral, so running it
meant touching `cli.py`'s hot path, which felt too risky to do without a live-verified module to
wire in first. User asked directly for the wiring: "SWE Bench based eval, and our existing eval
metrics all run [together] — tokens (broken down), time, files touched, etc."

Mechanism: `QuestionSet` gained an optional `swebench_eval_info` field (`None` for every ordinary
hand-authored question set; `swebench_dataset.instance_to_question_set` sets it to
`{dataset_name, split, instance_id, image}` for a dataset-native SWE-bench instance). `cli.py`'s
`_execute_run` checks for it after each variant's continuum completes (same point `harnessify_cost`
already gets attached) and, if present, runs a new `_run_variant_swebench_eval` per variant on its
own thread — builds that variant's real patch (`build_model_patch`, sandbox vs. its own pristine
fixture) and grades it via `evaluate_patch`, catching `SweBenchEvalError` rather than letting one
variant's Docker trouble take down the whole run. A `--swebench-eval/--no-swebench-eval` flag
(default on) reaches `run`, `batch`, and `elo` alike, since all three funnel through
`_execute_run` — one wiring point covers every entry point. A real bug caught immediately by the
new unit tests, before any live run: `swebench_eval` was used throughout `_run_variant_swebench_eval`
but never actually added to `cli.py`'s own `from . import (...)` block — would have been a bare
`NameError` the first time the code path actually ran.

Live-verified end to end: `traversal-compare run swe-bench-pilot --variant harnessified --no-viewer`
against the real, cached `psf__requests-2317` fixture. Real agent session ran, real Docker
container (`sweb.eval.psf__requests-2317...harnessified`) came up and ran for a couple of minutes,
and the resulting `result.json`'s `harnessified` variant carries all of it together: `metrics`
(tokens 8,110 split 5,291 in / 2,819 out, `wall_clock_seconds: 31.8`, `distinct_files_touched`,
`end_accuracy: 1.0`), `swebench_eval` (`resolved: true`, the real per-test FAIL_TO_PASS/PASS_TO_PASS
report), `harnessify_cost` (the cached prep session's own one-time cost), and `grade` (the
LLM-judge check, passed). No Docker containers left running afterward — the harness's own cleanup
handled that.

Also updated, since they were now stale: `SUITES.md` (removed the "not yet wired" caveat, corrected
the module docstring), `architecture.md`'s module table entry for `swebench_eval.py`, and
`skills/reporting-results/SKILL.md` (a `swebench_eval (resolved)` row for the checks-based report's
metrics table, with an explicit call-out that a disagreement between it and the LLM-judge check
belongs in Noteworthy — the two are now genuinely independent correctness signals on the same
variant, worth surfacing if they ever split).
