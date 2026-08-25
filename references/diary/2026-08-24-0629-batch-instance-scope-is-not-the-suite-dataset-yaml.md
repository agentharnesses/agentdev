---
description: User correction -- expanded suite-level dataset.yaml to 10 instances to mirror a requested batch expansion, then was told scope belongs per-batch only. Reverted the suite-level file to its original 3, grew the batch's own coverage instead, and confirmed live that --results-batch never depended on the suite-level list to begin with.
date: 2026-08-24 06:29 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — dataset.yaml revert + SKILL.md/SUITES.md updates not yet committed)
---

Asked to "expand the test to 10 questions from the benchmark." `psf/requests` only has 6 total
instances in SWE-bench Lite (3 already in `dataset.yaml`), so reaching 10 meant pulling from other
repos — asked the user how to pick them (stay small vs. go bigger/messier vs. a mix); answer was
"go bigger/messier," landing on 2 django + 2 sympy + 2 matplotlib + 1 scikit-learn (each repo's own
smallest-gold-patch instance, a rough complexity proxy) alongside the existing 3 `psf/requests`.

First pass: added all 10 to the suite-level `suites/swe-bench-pilot/dataset.yaml`. User's correction
immediately after: "the whole point of this is to apply it to the current batch... there are several
sources of truth, the scope in terms of id's is only relevant on a per-batch basis." Checked the
actual code rather than assuming — `swebench_dataset._resolve_instance_id` checks whichever
`dataset_config` it's handed, and `cli.py`'s `--results-batch` path hands it the *batch's own*
`dataset.yaml` (`swebench_results_store.get_dataset_config`), never the suite-level one at all. So
the suite-level edit wasn't just redundant, it was never load-bearing for the actual request —
confirmed live: `django__django-14534` resolves cleanly through the batch's own coverage
(`results add-instances`) with the suite-level file reverted back to its original 3 and never
mentioning it.

Reverted `dataset.yaml` to its original 3 `psf/requests` instances, with a comment explaining what
it actually is now (a small, stable template/allow-list for ad-hoc use — `traversal-compare list`,
a bare `run swe-bench-pilot/<id>` with no `--results-batch` — deliberately not where a batch's own
pilot scope lives). Grew the batch (`20260823T213625Z-d139e5a9`) to the real 10 via
`results add-instances`. Updated `SUITES.md` and `archiving-swebench-results/SKILL.md` to state the
rule explicitly (`results show-batch <batch-id>` is the one place to check a batch's actual
coverage, never the suite-level file) so this doesn't get relearned the hard way next time.

**Not yet done:** actually running/archiving the 7 newly-covered django/sympy/matplotlib/scikit-learn
instances — declared in the batch, not yet run.
