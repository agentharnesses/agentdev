---
description: Found and fixed a real documentation gap -- swebench_results_store.py's batch-archiving CLI (create-batch/add-instances/add-run/kpis/etc., plus run/batch's --results-batch flag) had been built and was in active use, but architecture.md and archiving-swebench-results/SKILL.md were never updated to mention it.
date: 2026-08-24 05:10 CDT
git:
  agentdev: d73c520 (dirty — traversal-compare submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — doc updates from this entry not yet committed)
---

User asked directly whether the "batches" work from earlier sessions had been fully documented in
harness/routing resources. Routing itself (`SKILLS.md`/`SUITES.md`/`HARNESS.md` chains) was intact
— `archiving-swebench-results/SKILL.md` and `results/RESULTS-STORE.md` were both properly linked
all the way up, confirmed via `reverse_disclose.py`. The real gap was content accuracy, not
structure: `references/architecture.md`'s module-to-file map had no row at all for
`swebench_results_store.py` (a real, 22KB module), and its `cli.py` row listed only
`run`/`batch`/`elo`/`embed-consistency`/`file-consistency`/`list`/`view`/`clean`/`validate` —
omitting the entire `results` command group entirely.

Digging into the actual CLI (`cli.py:1098` onward) also surfaced a feature that wasn't documented
anywhere at all, not even loosely: `create-batch`/`add-instances` give a batch its own declared
instance coverage (a `dataset.yaml` written under the batch's own directory), and `run`/`batch` both
accept `--results-batch <batch-id>` to resolve a question set against that batch's own coverage
instead of the suite-global `dataset.yaml` (`cli.py`'s `_resolve_question_set`). This is the real
mechanism for "work through SWE-bench across several sessions" — grow one batch's own instance list
over time via `add-instances`, run against it via `--results-batch`, archive into it via `add-run`.
None of this was in `archiving-swebench-results/SKILL.md`, which still only documented
`create-batch --reason` with no `--instance-ids`.

Fixed by updating `architecture.md` (new `swebench_results_store.py` row, extended `cli.py` row),
`archiving-swebench-results/SKILL.md` (documented `add-instances`, the `--results-batch` flow, the
new `add-run` coverage guard), and `running-a-test/SKILL.md` (pointer to `--results-batch` from the
`run` section). No code changed — this was purely a documentation-accuracy fix, verified against the
real `cli.py`/`swebench_results_store.py` source rather than guessed.
