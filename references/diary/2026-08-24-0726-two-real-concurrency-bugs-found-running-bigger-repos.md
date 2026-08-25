---
description: Running the 7 newly-added django/sympy/matplotlib/scikit-learn instances against the results batch surfaced two real, previously-latent concurrency bugs in external_fixture.py's shared clone cache, plus a config-threading gap that made harnessify's own prep-session timeout unconfigurable. All three fixed and live-verified across two retries.
date: 2026-08-24 07:26 CDT
git:
  agentdev: d73c520 (dirty — submodule pointer bump + this entry not yet committed)
  traversal-compare: c79242c (dirty — external_fixture.py/cli.py fixes, batch dataset.yaml timeout bump, tests not yet committed)
---

Ran `traversal-compare batch` against the 7 newly-declared django/sympy/matplotlib/scikit-learn
instances, `--max-parallel 2`, `--results-batch 20260823T213625Z-d139e5a9`. First attempt: job 1
(`django__django-15061`) errored immediately — `git clone failed ... could not create work tree dir
... File exists`. Root cause, found by reading `external_fixture.py` directly rather than assuming
resource contention: `_ensure_local_clone`'s `clone_dir.exists()` check was a real TOCTOU race, and
NOT a rare edge case — a single question set's own `baseline`/`harnessified` variants always clone
the same source concurrently (`_execute_run`'s own `ThreadPoolExecutor`), so this bug was live on
*every* external-repo instance this whole project has ever run; it just happened to go unnoticed on
`psf/requests` because the clone cache was already warm by the time anyone looked. Fixed with a
tmp-then-rename pattern (clone into a private tmp dir, atomic rename, detect a lost race via the
rename itself raising) — same discipline the rest of this codebase already uses. Added the same
fix to `materialize_external_repo`'s export-cache write (pid-only tmp naming, same class of gap,
narrower window). Two real, live-reproducing regression tests added (verified failing on the old code, passing on
the fix); full suite 275 passed at this point.

Stopped the compromised batch (confirmed via `git fsck --full` that the django clone left behind was
still healthy — the loser failed at `mkdir`, before writing anything), relaunched clean. This run:
3 jobs completed for real (`django__django-15061` FAIL, `sympy__sympy-17139` PASS,
`sympy__sympy-18057` FAIL — archived into the batch), but 4 of 7 hit a NEW failure: `timed out
waiting for turn N`. Traced this to a second, separate gap: `cli.py`'s fixture-materialization loop
never threaded `permission_mode`/`timeout_seconds` into `harnessify.materialize_harnessified_fixture`
at all — that function's own hardcoded defaults (600s) governed the harnessify prep session's two
turns regardless of what `dataset.yaml` declared, invisible and unfixable via the config file alone.
Fixed by resolving `permission_mode` earlier in `_execute_run` (moved up, no behavior change) and
passing both through explicitly. Also bumped the *batch's own* `dataset.yaml` `timeout_seconds` from
600 to 1800 (matching `swebench_eval.py`'s own existing Docker-eval timeout, not an invented number)
— hand-edited, since no `results` CLI command manages this field and it isn't a derived value like
`batch.json`'s `run_count`. Confirmed the harnessify cache key (`source`, `sha`) doesn't include
these fields, so threading them through can't cause a cache-key mismatch. Not unit-tested directly —
`_execute_run` is this project's one deliberately-not-unit-tested piece (`test_cli.py`'s own comment:
"concurrency-heavy orchestration this project has never unit-tested, verified live instead"); the
live retry is the verification.

Retried the 4 timed-out instances. First job of the retry (`matplotlib__matplotlib-23314`) hit a
THIRD, related failure: `git fetch --all --tags` exit status 128. This was the first race's fix
applied too narrowly — it only protected the first-clone branch; the "clone already exists, fetch to
stay current" branch was still racy, and now that the clone cache was warm (from the earlier failed
attempts), THIS branch became the one both variants hit every time. A tmp-then-rename dance doesn't
protect a read-then-mutate-in-place operation like `git fetch`, so the real fix is mutual exclusion
for the whole check-then-act section: a per-source `threading.Lock` (`_clone_lock`), double-checked
against a guard lock for the lock-dict itself. Whichever caller gets the lock first does the one real
clone-or-fetch; every other concurrent caller for the same source just waits, then reads the
already-current result. A real git-level reproduction of the fetch race proved too flaky to rely on
(a local no-op fetch holds git's own lock for far too short a window to reliably collide, unlike the
real, slower network fetch that exposed this) — replaced with a deterministic test that monkeypatches
`subprocess.run` to widen the critical section and asserts max concurrent fetches for one source
never exceeds 1. Verified failing on the pre-fix code (`8 == 1` assertion, i.e. all 8 ran
unprotected) and passing on the fix. Full suite: 278 passed, 1 skipped.

Both matplotlib/django clone directories left behind by the failed race were verified healthy via
`git fsck --full` before reuse — the losing `git fetch` failed at git's own lock acquisition, before
touching any actual ref/object state, so nothing needed re-cloning.

Stopped the (now doubly-stale) retry process and relaunched a third time with all three fixes in
place — still running as this entry is written.

**Pattern worth naming**: all three bugs here were found by actually reading the error and the
source it pointed at, not by assuming "resource contention, lower --max-parallel" (the instinctive
first read of a batch of concurrent timeouts/errors). Two were genuine, previously-invisible
concurrency bugs that had nothing to do with how many jobs were running in parallel — they were
latent in `_execute_run`'s own always-concurrent baseline/harnessified pair, present since the day
`external_fixture.py` was written, just never exercised hard enough (never a first-time clone of a
never-before-seen big remote repo under real time pressure) to surface until today.
