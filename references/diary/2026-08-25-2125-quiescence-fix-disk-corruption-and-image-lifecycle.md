---
description: Follow-up to the real-Docker-environment entry -- a deeper concurrency fix was
  needed beyond the first one (Claude Code's startup renders in multiple discrete bursts, so
  a single non-empty poll wasn't enough; fixed with a genuine quiescence/settle-window check).
  Then a live batch run filled the host disk with per-instance SWE-bench images (some several GB
  each) until Docker's own containerd metadata store corrupted -- docker rm/rmi/system df all
  started failing with I/O errors. Recovered by freeing host cache space, force-restarting Docker
  Desktop, and manually removing 58 stale images (~150GB). Built a lasting fix: cli.py's batch
  runner now prefetches each job's image a few jobs ahead and tears it down right after that job's
  run is archived. Growing the batch to the full 50-instance set (matching the earlier prompt-based
  batches) surfaced two more real bugs -- an in-container auto-updater doomed to fail and blowing
  startup timeouts, and `close()` not reliably stopping the remote container -- both root-caused
  and fixed, confirmed clean on retry. Batch finished at 50/50, after also catching a self-inflicted
  archiving gap (two completed runs never actually got archived).
date: 2026-08-25 23:45 CDT
git:
  agentdev: b44ae87 (dirty -- this entry not yet committed)
  traversal-compare: e57ba0d (dirty -- claude_runner.py quiescence fix + DISABLE_AUTOUPDATER +
    container-name/stop fix, cli.py prefetch/teardown + batch scheduling, agent_image.py
    image_tag()/remove_agent_image(), question_sets.py, swebench_dataset.py,
    suites/swe-bench-pilot/dataset.yaml, batch 20260825T150201Z-9dc0ca84 now complete at
    50/50 runs, not yet committed)
---

Continuation of `2026-08-25-1207-real-docker-environment-built-and-concurrency-bug-found.md`,
same day. That entry closed with the concurrency hang apparently fixed (require at least one
non-empty poll before trusting "no gate matched" as ready) and one real instance verified clean.
Three more things happened before the day's real-environment batch actually finished: a second,
deeper concurrency bug: `_resolve_startup_gates` needed a genuine quiescence window, not read from
a diary -- a live disk-corruption incident that stopped the batch cold, and the architectural fix
that came out of it, and a small operational mistake caught fast.

**The first fix wasn't enough.** Running the grown batch (astropy/matplotlib/flask/requests
instances, `20260825T150201Z-9dc0ca84`) with two concurrent containers, sessions sat at 0
transcript lines for 150s+ despite alive containers with real CPU/memory usage and an already-
logged "gates resolved" message -- the fix from the previous entry had visibly done its job
(waited for real output before declaring ready) and the session still never produced a transcript.
Ruled out repo size (astropy has fewer files than Django, which had run cleanly). The real cause:
Claude Code's startup renders in multiple discrete bursts -- the trust-folder gate, then a
"Tips for getting started" welcome box, then the actual ready prompt -- so a single non-empty poll
can land on any of those bursts, not necessarily the last one. The first fix only proved *some*
output had arrived, not that the terminal had finished rendering. Fixed by requiring the pty to go
genuinely quiet (no new content) for a full 1.5s settle window after real output was last seen,
before trusting "no gate matched" as ready -- the same stability-window discipline `_wait_for_turn`
already used for turn completion, now applied to startup too. Live-verified: zero stalls across the
rest of that batch's 9/20 jobs and 40+ minutes of concurrent runs.

**Then the batch filled the host disk.** Partway through growing the batch to its full 20
instances, routine monitoring turned up `df -h /` showing 872MB free out of 926GB -- Docker
Desktop's virtual disk (`Docker.raw`, a sparse file on the host) had grown to 145GB of real usage
by simply accumulating: every SWE-bench instance ships its own multi-GB pre-built environment
(matplotlib/astropy images ran 10+GB each, most others 3-5GB), and the wrapper-image cache from
`agent_image.py` -- correct on its own terms, content-addressed and non-duplicating -- was still
adding one more full-size tagged image per instance on top, with nothing ever removing either once
a job was done. One job (`matplotlib__matplotlib-25433`) failed outright: `docker build` hit
`input/output error` reading a content blob mid-pull. Worse, this wasn't just space starvation --
`docker rm`/`docker rmi`/`docker system df` all started failing too, with the SAME class of error,
eventually surfacing as a write failure against containerd's own metadata bolt db
(`io.containerd.metadata.v1.bolt/meta.db`). A chicken-and-egg problem: cleaning up Docker requires
Docker to have room to write its own metadata, which requires the exact space the cleanup was
supposed to free.

**Recovery, in order:** freed ~15GB of real host space via standard, fully-regenerable app caches
(browser caches, pip/Homebrew/pnpm/playwright/node-gyp) -- did not touch the disk this way alone;
containerd's metadata db was genuinely corrupted, not just space-starved, so `docker rm`/`rmi` still
failed with the same write error even with headroom back. Force-restarted Docker Desktop (a
graceful `quit app "Docker"` hung for 2.5 minutes, needed `pkill -9` against the Docker Desktop and
backend processes) -- the restart itself repaired the corruption; `docker images`/`system df` both
came back clean afterward. Then manually removed 58 images no longer needed for the run in
progress (11 completed-instance images from this batch, 6 from the original sanity-check set, 24
fully unrelated images left over from other days' comparison work -- xarray, scikit-learn, sympy,
pytest-dev, pylint-dev, sphinx-doc, an old `requests` instance), reclaiming roughly 150GB and
leaving 33GB+ free, while deliberately leaving alone a handful of small images from unrelated
projects already on the machine (`groundxonboarding-*`, `valantor-fe-test`, `k8s-minikube`) that
were never part of this work.

**The lasting fix: image lifecycle, not just cleanup.** The user's framing, verbatim: "we should
scrap all of the images we're not using in this test... adjust the process so it prepares the next
batch while the current one is processing, then deletes the images that correspond with a
completed one." `agent_image.py` gained two small public functions: `image_tag(base_image)` (the
wrapper tag a build *would* produce, without touching Docker -- lets a caller reason about an
image's name for teardown purposes without paying for an `image inspect` round trip) and
`remove_agent_image(base_image)` (best-effort `docker rmi` of both the wrapper tag and the base
image, swallowing "still in use"/"not found" rather than raising -- cleanup must never sink a run
whose result already succeeded). `cli.py`'s `batch_cmd` now resolves every job's target image once,
up front, into a `job_images` list plus a refcount (so a repeated `--times` or an accidental
same-instance duplicate doesn't get torn down while another queued job still needs it); each job,
at the moment it starts, kicks off a prefetch for the job `max_parallel` slots ahead of it, and once
a job's own run is fully archived (continuum *and* grading both done -- `swebench_eval.py` re-pulls
the base image too, so tearing down any earlier than that would just force a re-pull), its image
gets released and, if nothing else needs it, removed immediately.

One bug in that first implementation, found before it shipped: the prefetch call was synchronous,
so a job's own thread could block on building a *different*, later job's image before even starting
its own work -- the opposite of the intent. Only harmless in the run that surfaced it because that
particular image happened to already be cache-hit (built during an earlier, killed attempt at the
same batch). Fixed by backgrounding the prefetch in a daemon thread instead.

**Confirmed live, end to end.** Relaunched the batch's remaining 10 instances (the 9 truly
unfinished plus `matplotlib-25433`, since its earlier failure was the disk corruption, not a real
result) under the new code. All 10 completed; every single one's images were confirmed gone from
`docker images` within seconds of that job's run being archived -- checked by name after each
completion, not assumed. Disk stayed flat around 50-57GB free the entire run (never dipped, in
several cases ticked back up slightly as jobs completed); total image count stayed in the
teens/low-twenties throughout instead of climbing toward the 80+ that caused the original crisis.
The whole batch (`20260825T150201Z-9dc0ca84`) now stands at 26/26 archived runs.

**One small operational mistake, caught fast.** The first relaunch attempt after the Docker Desktop
restart used a fresh shell that never had `CLAUDE_CODE_OAUTH_TOKEN` re-exported into it (shell
state doesn't persist across separate tool invocations, and the restart had cleared whatever the
prior shell held) -- its first 4 jobs failed instantly with a clear, correct error
("`CLAUDE_CODE_OAUTH_TOKEN` isn't set") rather than anything confusing. Fixed by re-exporting from
the token file inline (`export CLAUDE_CODE_OAUTH_TOKEN="$(cat <path>)"`, value never printed to any
tool output or this conversation) and relaunching for real. Notably, the failed attempt's prefetch
calls still ran successfully before each job's token check failed (prefetch doesn't need the
token) -- those builds weren't wasted, they were still sitting in Docker's cache when the real
relaunch started, which is part of why the real relaunch's own jobs 0 and 1 started instantly.

**Interim numbers at 26 runs** (before growing further): pass_rate=65%, both variants at 69%
resolved, harnessified ~33% fewer mean tokens than baseline. Of that sub-batch's 10 instances: 6
passed, 4 failed (`matplotlib-25498`, `flask-4992`, `flask-4045`, `requests-3362`), no obvious
repo-specific clustering.

**Growing to full 50-instance parity surfaced two more real bugs.** The user asked to add the 24
instances that appear in both earlier prompt-based "50-instance" batches
(`20260824T193834Z-5b852589`, `20260825T051502Z-af3ba59f` -- confirmed identical instance lists)
but had never run under the real-Docker-environment architecture: `psf__requests-863`,
`pydata__xarray-4094`, 2 pylint-dev, 3 pytest-dev, 5 scikit-learn, 2 sphinx-doc, 10 sympy. None of
their base images had ever been pulled/built before, so this run exercised cold image paths the
first 26-instance run mostly hadn't.

**Bug 1: Claude Code's in-container auto-updater is doomed to fail, and its failure blows the
startup timeout.** Three instances hard-errored after exhausting all 3 retry attempts, with two
different-looking failure signatures: `scikit-learn__scikit-learn-11040` timed out resolving
startup gates; `scikit-learn__scikit-learn-13142` and `sphinx-doc__sphinx-7738` crashed
(`claude exited mid-session (rc=1)`) right after the pty was declared ready. Tracing
`scikit-learn-13142`'s full 3-attempt retry history end to end showed both signatures are the SAME
bug: every attempt's terminal was showing, verbatim, `✘ Auto-update failed: no write permission to
npm prefix · Run claude doctor` -- Claude Code trying to auto-update on startup, failing because
the wrapper image's npm-global install happens as root during `docker build` (before the
Dockerfile switches to the non-root `agent` user), so `agent` can never write there. Attempts 1 and
2 timed out with that exact text as the last-seen output (the auto-update spinner-then-failure
sequence kept the pty non-quiescent long enough to blow the fixed startup-gate timeout); attempt 3
crashed outright at the same point. Confirmed live via `strings` against the host's own installed
`claude` binary that `DISABLE_AUTOUPDATER=1` is a real, documented env var
(`~/.claude/settings.json`'s `env` block, or a bare env var, both turn off background auto-updates).
Fixed in `claude_runner.py`'s `_build_container_argv`: writes `DISABLE_AUTOUPDATER=1` into the same
env-file the OAuth token already goes through. This is the *correct* fix, not just a timing
one -- `agent_image.py` already pins the container's `claude_version` to match the host specifically
so a comparison run's containerized and host-run CLI are the same build; letting the container
auto-update on its own would have silently defeated that pinning even when it didn't outright crash
the session.

**Bug 2: `InteractiveSession.close()` doesn't reliably stop the container.** Found by chance while
investigating an unrelated flat-transcript check: `docker ps` showed several already-archived (and
one already-hard-errored) jobs' containers still `Up`, minutes after their own `_execute_run` had
already returned successfully. `close()` was only calling `self._proc.terminate()`/`.kill()` on the
*local* `docker run` client process (spawned with `start_new_session=True`); terminating that
client does not reliably propagate a stop to the daemon-managed container, and a `.kill()` on
graceful-terminate timeout gives the client no chance to forward anything at all. Manually cleaned
up 6 leaked containers (`sphinx-8721` x2, `sphinx-7738` x2, `scikit-learn-14092` x2) as a courtesy
sweep. Fixed properly: `_build_container_argv` now passes an explicit `--name` (derived from
`session_id`, returned as a third tuple element) alongside the existing `--rm`, and `close()` now
also issues `docker stop --timeout 5 <name>` directly against the daemon, independent of whatever
the local client process does or doesn't do -- a successful stop plus `--rm` guarantees removal,
not just a stop. Best-effort: swallows "no such container" (the common case, when the client-side
terminate already worked).

**Both fixes confirmed clean on retry.** Relaunched the 3 hard-errored instances
(`scikit-learn-11040`, `scikit-learn-13142`, `sphinx-doc-7738`) under the fixed code. All 3
resolved for real this time (2 PASS, 1 FAIL -- no more ERRORs), `grep -c "Auto-update failed"`
stayed at 0 for the entire retry log, and after archiving each one its containers were confirmed
completely gone from `docker ps -a` (not even lingering as `Exited`) -- both bugs verified fixed,
not just patched-and-hoped.

**A third, self-inflicted gap, also caught before it mattered.** After what looked like the final
archive (48/50), a direct count against the batch's own declared 50-instance `dataset.yaml` showed
only 48 runs actually present on disk -- `pytest-dev__pytest-5227` and `pytest-dev__pytest-6116`
had both genuinely completed (PASS, seen in the log early in the run) but were never archived, most
likely dropped during a monitoring-cycle handoff earlier in a very long session. Found by checking
ground truth (declared instance list vs. archived runs' own `question_set_id` fields) rather than
trusting the running "N/50" tally, and fixed by finding their real run-ids under the temp runs
directory and archiving them directly. Worth remembering: a long chain of self-reported counts is
exactly the kind of thing that should get a ground-truth check before calling something done.

**Final numbers, full batch** (50 runs, baseline vs. harnessified, all graded against the real
per-instance Docker eval image):

```
overall: pass_rate=72%  tokens mean=43,638 (in=32,840 / out=10,798)  wall_clock mean=214.6s
baseline       n=50  resolved_rate=80%  tokens mean=47,752  wall_clock mean=218.5s
harnessified   n=50  resolved_rate=76%  tokens mean=39,523  wall_clock mean=210.6s
harnessify_prep (one-time, mean): tokens=222,856  wall_clock=618.0s
breakeven vs. baseline: tokens=27.1 queries  wall_clock=78.6 queries
```

Per-repo pass rate across the full 50: mwaskom 1/1, astropy 2/2, pylint-dev 2/2, pydata 1/1 (all
100%, small n); scikit-learn 4/5 (80%); sympy 8/10 (80%); psf 3/4 (75%); matplotlib 3/4 (75%);
pytest-dev 2/3 (67%); django 8/13 (62%, the largest single group); sphinx-doc 1/2 (50%); pallets
1/3 (33%, the weakest). No single repo dominates the failure count -- django and pallets/flask
account for most of it simply by instance count, not a distinct pattern. Resolved rate moved from
identical-at-26 (69%/69%) to a real gap at 50 (baseline 80% vs. harnessified 76%) -- worth a fresh
McNemar pass rather than reading anything into the raw gap directly, same discipline as the
morning's statistical correction; not yet run.

**Not yet done:** committing the day's code (`claude_runner.py`, `cli.py`, `agent_image.py`,
`question_sets.py`, `swebench_dataset.py`, `dataset.yaml`, this and the prior three diary entries);
a fresh statistical pass (McNemar/Wilcoxon) comparing this real-environment batch against the
strengthened-prompt batch now that both have a real completed n=50.
