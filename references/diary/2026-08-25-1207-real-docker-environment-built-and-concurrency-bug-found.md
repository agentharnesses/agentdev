---
description: Implemented the real-Docker-environment architecture change (agents run inside the actual per-instance SWE-bench eval image, prompt_prefix dropped entirely) and spent most of the day chasing an intermittent concurrent-container-startup hang before the user's own question -- "why are we in the dark?" -- prompted adding real progress logging, which immediately surfaced the actual bug: InteractiveSession's startup-gate resolution could declare a session ready having observed zero real output. Fixed; a full concurrent baseline+harnessified run then completed cleanly end to end.
date: 2026-08-25 12:07 CDT
git:
  agentdev: b44ae87 (dirty -- this entry not yet committed)
  traversal-compare: e57ba0d (dirty -- agent_image.py, claude_runner.py container support + bugfix,
    cli.py wiring + retry logic, suites/swe-bench-pilot/dataset.yaml, new batch
    20260825T150201Z-9dc0ca84, not yet committed)
---

Continuation of `2026-08-25-0841-statistical-correction-and-swebench-canonical-methodology.md`,
same day. That entry ended with the user choosing the architecturally-aligned fix over a prompt
tweak: give agents the real per-instance SWE-bench Docker eval image as their own working sandbox,
matching SWE-agent's own canonical setup, instead of continuing to suppress self-verification via
`prompt_prefix`.

**Planned properly first.** Explored `external_fixture.py`, `claude_runner.py`,
`harnessify.py`, `swebench_eval.py`, `question_sets.py`, `cli.py` to scope the change, found a real
blocker before writing any code: the host's own `claude` binary is a native macOS Mach-O
executable -- it cannot run inside a Linux container at all. Wrote a full plan
(`EnterPlanMode`/`ExitPlanMode`) covering a Phase 0 feasibility spike before any pipeline code, got
it approved.

**Phase 0 spike, live-verified:** built a wrapper image (`FROM <real swebench eval image>` + Node
+ pinned `@anthropic-ai/claude-code`) on top of a real per-instance image, confirmed `claude`
authenticates inside the container via `claude setup-token`'s output propagated as
`CLAUDE_CODE_OAUTH_TOKEN` (user's own chosen mechanism), confirmed the interactive pty-driven mode
this project depends on works through `docker run -it` unmodified, confirmed a real Bash tool call
against the actual pre-installed dependencies works (a real `pytest` run inside a genuinely
different `testbed` conda env, self-corrected by the agent after noticing the base env couldn't
even import the package). Two real fixes the spike surfaced beyond the plan: `bypassPermissions`
refuses to run as root (every SWE-bench image is root by default; added a non-root `agent` user),
and `~/.claude.json` (a sibling file, not inside `.claude/`) has to be mounted too or the container
shows a first-run onboarding wizard instead of the normal trust-folder gate.

**Implementation**, once the spike passed: `agent_image.py` (new module, builds/caches the wrapper
image per base image, content-addressed by `(base_image, claude_version)`), `claude_runner.py`
gained `container_image`/`_build_container_argv` (docker-wraps the existing argv, mounts `cwd` and
`$HOME` at identical paths so every existing host-side path convention keeps working unmodified),
`question_sets.py`/`swebench_dataset.py` gained `use_agent_container`, `cli.py` wired it through
with a fail-fast check for the OAuth token. `suites/swe-bench-pilot/dataset.yaml` updated:
`use_agent_container: true`, `prompt_prefix` removed entirely on both variants (the whole point --
stop fighting the model's normal Test Execution instinct now the environment genuinely supports
it). New batch `20260825T150201Z-9dc0ca84`, seeded with the same 6-instance sanity-check set as the
strengthened-prompt batch for direct comparability.

**Then a very long concurrency debugging odyssey.** A single containerized session worked cleanly
every time from the start. Two concurrent ones (baseline + harnessified, always launched together)
hung indefinitely -- near-zero CPU, no transcript ever created. Chased this for hours with a real
bisection discipline, not guessing: isolated `.claude.json` to a private per-session copy (didn't
fix it); isolated the whole `~/.claude` mount to a private synthetic `$HOME`
(`claude_runner.prepare_container_home`, still didn't fix it); tested two distinct OAuth tokens
(flaky -- sometimes one side fully succeeded, sometimes both failed differently); ruled out disk
space (a real, separate crisis found mid-investigation -- Docker had quietly eaten the host down to
2.8GB free, cleaned up ~5GB of build cache + old wrapper images, kept the 50 real SWE-bench eval
images per user's choice) and memory (`docker stats` showed both containers at ~180MB of 7.65GB the
whole time) and network (raw `/proc/net/tcp` inspection showed all sockets `ESTABLISHED`, zero
bytes queued either direction -- not actually stuck waiting on the API). Every isolation attempt
either failed to fix it or produced inconsistent, non-reproducible results.

**The actual bug, found by finally doing what the user asked instead of guessing from outside
symptoms.** User's question, verbatim: "Is there a reason we're in the dark? If we're injecting
the CLI, can't we check some simple logic that runs and analyzes the CLI for issues?" -- a fair,
overdue critique; every diagnostic up to that point inferred state indirectly (transcript file
existence, `docker stats`, raw `/proc` inspection) rather than the code just reporting what it was
doing. Added `_progress()`, plain always-on stderr logging (not the `logging` module -- this
project uses none of it; kept off the pty and off stdout so it can't corrupt gate detection or
collide with `click.echo`), instrumented `_resolve_startup_gates` and `_wait_for_turn`. Writing
that instrumentation surfaced the bug directly, before ever running it: `_read_available()` returns
`""` (not `None`) whenever nothing arrives within its own 1.5s poll window but the pty is still
open -- a completely normal outcome for a container that's simply slow to render its first frame.
The *original* `_resolve_startup_gates` treated "no gate matched in what's been seen so far" as
sufficient grounds to declare the session ready -- on a session whose very first 1.5s poll happened
to return empty (zero real output observed, not a sign of anything wrong), it would be declared
ready anyway. `send()` would then write the real prompt into a screen that hadn't finished booting.
This explains every symptom the whole day chased without ever finding a root cause: purely a race
on which of two concurrent containers happened to render *anything* within its own first 1.5s poll;
zero correlation with filesystem sharing, tokens, disk, memory, or network, because none of those
were ever the actual variable. Fixed by requiring at least one real, non-empty poll before "no gate
matched" is trusted as "ready."

**Live-verified immediately after the fix.** Killed the in-flight old-code smoke test, rebuilt with
the fix, relaunched the same real instance (`django__django-16379`, real Docker environment, both
variants concurrent). Full progress log this time instead of a black box: both sessions correctly
waited through 4 polls before the first real output arrived and only then declared ready (the fix
visibly doing its job); both proceeded through real, growing transcripts with no hang; harnessified
completed its priming turn (21.7s) then its real step (42.5s); baseline completed its own real step
(53.1s). `Run 20260825T170300Z-a0f6de97: PASS` -- both variants' patches resolved for real
(`swebench_eval` confirmed, not just the LLM-judge path), no self-verification needed for this
particular instance (it's historically been a fast, simple one), no manual intervention, no retry
even needed. Docker cleanup confirmed clean (`docker ps` empty of anything from this run) after.

**A second, complementary fix landed alongside this one**: `cli.py`'s `_run_variant_continuum`
now retries a fresh containerized start (session + priming + first step) up to 3 times on
`InteractiveSessionError`, discarding the failed attempt's synthetic home and generating a fresh
one each retry -- added as the pragmatic mitigation *before* the real bug was found, kept as a
defense-in-depth safety net now that it has been (some genuine transient flakiness -- a slow
network blip, a briefly-contended Docker daemon -- is still plausible even with the real bug fixed,
and retrying costs nothing when it isn't needed).

**Not yet done:** running the new batch (`20260825T150201Z-9dc0ca84`) for real beyond this one
manual smoke-test instance; committing the new code (`agent_image.py`, `claude_runner.py`,
`cli.py`, `question_sets.py`, `swebench_dataset.py`, `dataset.yaml`); a proper statistical
comparison (McNemar/Wilcoxon, same discipline as this morning's correction) between this real-
environment batch and the strengthened-prompt batch once enough real instances have run.
