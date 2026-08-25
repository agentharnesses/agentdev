---
description: Asked which model was actually being used -- discovered no invocation anywhere in this pipeline ever set --model, so every run silently rode whatever the claude CLI's own default resolved to (confirmed live: claude-sonnet-5). Added an explicit, batch-level model field (QuestionSet.model, threaded through claude_runner/harnessify, folded into harnessify's own fixture cache key) and pinned it on both the suite template and the currently-open batch.
date: 2026-08-24 18:56 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — model-threading code/tests/dataset.yaml changes not yet committed)
---

User asked what model was actually running these swe-bench-pilot sessions. Checked directly rather
than assume: `claude_runner._build_interactive_argv` never sets `--model`, and neither does
`harnessify.py`'s own prep session or `swebench_judge.py`/`grading.py`'s judge calls -- every
invocation in this whole project has always ridden whatever the `claude` CLI's own default resolved
to at the time. Confirmed live, same invocation style this project uses (`claude -p ... --setting-
sources "" --strict-mcp-config`): the primary answering model is `claude-sonnet-5` (a small
ancillary/routing call also touches `claude-haiku-4-5`, not the model doing the actual work).

User's ask, following that: lock the specific model for the rest of the current batch, and make it
an explicit, defined field at the batch level going forward, not an implicit CLI default.

Threaded `model: str | None = None` through the same path `permission_mode`/`timeout_seconds`
already use, end to end:

- `QuestionSet.model` (`question_sets.py`) -- read from a hand-authored `question-set.yaml`'s own
  `model` key (`load_question_set`), or from `swebench_dataset.py`'s `dataset_config.get("model")`
  (`instance_to_question_set`) for the dataset-native path.
- `claude_runner._build_interactive_argv`/`InteractiveSession` -- a `model` param that adds
  `--model <model>` when given, `None` (the default) preserving the old implicit-default behavior
  exactly for every caller that doesn't pass one.
- `cli.py`'s `_run_variant_continuum` passes `question_set.model` into every `InteractiveSession`
  it opens; the fixture-materialization loop passes it into `harnessify.materialize_harnessified_fixture`
  too, for the same reason `permission_mode`/`timeout_seconds` were threaded into that same call
  earlier today (2026-08-24-0726 entry) -- the prep session is a real `claude` invocation too, and
  should honor the same pin the answering sessions do.
- `harnessify._cache_key` now folds `model` in alongside the existing `source`/`sha`/skill-version-
  hash -- same reasoning as the skill-version hash already there: a fixture harnessified under one
  model shouldn't be silently handed to a caller that just pinned a different one.
  `model=None`/unset produces the exact same key the function always returned before this field
  existed, so every fixture already cached under the old, implicit-default behavior stays valid
  under its own key -- only a *newly pinned* model forces a fresh, model-pinned prep session, for
  exactly the instances that haven't been run under that pin yet.

New tests for every hop (argv flag present/absent, cache-key sensitivity including the
none-still-matches-none backward-compat check, `materialize_harnessified_fixture` actually passing
`model` through to the session, both `dataset.yaml`-loading paths threading it into `QuestionSet`).
Full suite: 295 passed, 1 skipped (up from 288 -- 7 new tests, zero regressions, since every new
parameter defaults to `None` and every existing call site that doesn't pass one gets byte-identical
argv/cache-keys to before).

Pinned `model: claude-sonnet-5` in two places: the suite-level template
(`suites/swe-bench-pilot/dataset.yaml`, so a future `create-batch` inherits it automatically --
`dict(template)` in `results_create_batch_cmd` copies every key, not a hardcoded list, confirmed by
reading that code rather than assuming) and the currently-open batch's own dataset.yaml
(`20260824T193834Z-5b852589`, hand-edited, same precedent as that batch's earlier `timeout_seconds`
edit -- no `results` CLI command manages this field). Live-verified end to end: both configs'
`model` value reaches a real built `QuestionSet.model` for both variants, and a real
`claude -p ... --model claude-sonnet-5` call succeeds.

**A real discontinuity this creates, disclosed rather than hidden**: every run already archived in
`20260824T193834Z-5b852589` (all 30) predates this field and has no `model` recorded in its own
`result.json` -- they ran under whatever the CLI's implicit default was at the time, which is very
likely `claude-sonnet-5` too (nothing in this session changed CLI version or config between then and
now) but isn't *recorded* as such. Every run archived into this batch from now on will have
`--model claude-sonnet-5` explicit and will re-harnessify fresh wherever the cache doesn't already
have a model-pinned entry.

**Not yet done**: `swebench_judge.py`/`grading.py`'s own LLM-judge calls still don't pin a model --
out of scope for this request (they're diagnostic, not the measured treatment), but the same
implicit-default risk applies to them too if it ever matters for judge-verdict reproducibility.
