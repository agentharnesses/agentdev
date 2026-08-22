---
description: Recovered mid-crash work (swebench_dataset.py, dataset-native swe-bench-pilot), then built swebench_eval.py's execution-based grading -- patch generation real-verified, the actual Docker harness call still unverified.
date: 2026-08-21 18:10 CDT
git:
  agentdev: 2a407d3
---

A crash lost the session, not the work -- everything was still on disk, uncommitted. Recovered it
by diffing against HEAD: `traversal-compare` had a finished, tested (`195 passed`) feature making
`swe-bench-pilot` dataset-native -- `swebench_dataset.py` loads a real SWE-bench Lite instance
straight from its HuggingFace parquet export instead of a hand-authored `question-set-N.yaml`,
building a `QuestionSet` in memory with an LLM-judge check auto-generated from the instance's real
`patch` field. The old single-instance `question-sets/question-set-1.yaml` was already deleted in
favor of `suites/swe-bench-pilot/dataset.yaml`'s `instance_ids` allow-list, and `cli.py` already had
a shared `_resolve_question_set()` trying the dataset path before falling through to the ordinary
one. `vendor/metaskill`'s `harnessify/SKILL.md` also had a small uncommitted tweak (re-validate
after fixing flagged issues, not just once).

What was missing: the module's own docstring promised a `swebench_eval.py` for execution-based
grading (real FAIL_TO_PASS/PASS_TO_PASS via the actual SWE-bench Docker harness, alongside the
LLM-judge fallback) and referenced a diary entry for the reasoning that was never written. Built
`swebench_eval.py` to close that gap:

- `build_model_patch(fixture_root, sandbox_root)` -- turns a sandbox's plain, `.git`-less file
  state into a real `git apply`-able unified diff against the pristine fixture. `git diff
  --no-index` was the obvious first choice and wrong: it embeds `fixture_root`'s/`sandbox_root`'s
  own directory names into the patch header instead of tree-root-relative paths, since the two
  directories are never named the same thing. Works instead by `git init`-ing a throwaway repo on
  a copy of the pristine tree, committing it, mirroring the sandbox's actual files on top
  (including deletions), and diffing against that one commit.
- `evaluate_patch(...)` -- shells out to the real harness (`python -m
  swebench.harness.run_evaluation`, not reimplemented) and reads back its own per-instance
  `report.json` (`get_eval_report`'s shape, keyed by instance_id, `resolved: bool` the field that
  matters). Argv, predictions-file shape, and report layout transcribed from the real harness's
  source and evaluation guide -- not guessed.

Real bug caught by testing `build_model_patch` against a real round-trip through `git apply`
(matching this project's usual real-git-plumbing test discipline, not mocked): the first version
used `git diff HEAD` alone, which silently dropped every newly-added file from the patch, since
diffing against a commit only considers already-tracked paths -- a file the mirror step just wrote
is untracked until staged. Fixed with a `git add -A` before the diff. Deletions were already
handled correctly (tracked files removed from the working tree show up fine). Full round-trip test
(modify + add + unchanged + a dedicated deletion case) now applies the generated patch to a fresh
checkout of the fixture and asserts the result is byte-identical to the sandbox.

Deliberately did NOT wire `evaluate_patch` into `cli.py`'s live `_execute_run` path this session.
Two real constraints, not just caution: sandboxes are ephemeral (`sandbox.delete_sandbox` runs
right after grading, per `methodology.md`), so execution-eval would have to run inline in the hot
path every `run`/`batch`/`elo` invocation takes for a dataset-native suite -- and this environment
has Docker installed but not running, and the real `swebench` package isn't installed
(`pip install -e '.[swebench]'` not yet done). `evaluate_patch`'s own harness invocation and report
parsing are only unit-tested against a mocked `subprocess.run`, matching how `test_harnessify.py`
treats `_run_ahar_init`/`_run_ahar_validate` -- correct by construction, not by having watched it
happen. Modifying every run's hot path on an unverified assumption would break this project's own
"live-verify before trusting" discipline the harder way, by shipping something that only fails the
first time someone actually runs it. Next real step: `pip install -e '.[swebench]'`, start Docker,
and run `evaluate_patch` for real against `psf__requests-2317`'s own harnessified-variant patch
before either wiring it in or reporting an execution-based result as real.
