---
description: Ran efficient-exploration live, found the vscode:// viewer popup was a stale-extension dead end (worked around via dev-host), traced why with-harness lost on tokens/time to the metaskill never being invoked, and designed (not yet implemented) a separate-cost "harness priming" step to address it.
date: 2026-08-19 20:32 CDT
git:
  agentdev: 9c5e942 (dirty — this entry's own changes, plus doc edits below, not yet committed)
  ahar-visualizer: 91ad3b9
  traversal-compare: 079fe64 (dirty — architecture.md/running-a-test/viewing-results SKILL.md edits not yet committed)
---

## Live-viewer popup: a known gap, not a new bug

Ran `traversal-compare run efficient-exploration --variant both` live. Got a macOS "open this
URL?" popup instead of panels. Root cause, confirmed by diffing compiled JS: the real main
window's installed `ahar-visualizer` (v0.2.0, installed 00:54am) predates the `openTree` URI
handler (`registerUriHandler`, added in commit `7acfe35`, 11:46am same day) — `grep
registerUriHandler` finds it in current `out/extension.js`, not in the installed copy's. This is
exactly what `2026-08-19-1145`/`2026-08-19-1230` already documented and left unfixed for the
main-window case. `open "vscode://...openTree?..."` fires the OS confirmation dialog regardless of
whether anything's listening, then silently no-ops against a stale install.

Couldn't fix by reinstalling: this session's shell lives inside that exact main window's
integrated terminal (confirmed via `ps` ancestry), and `ahar-visualizer-dev-workflow.md`'s hazard
note is explicit that *reloading* that window (not just closing it) kills the session mid-work.
Per the user's steer, worked around instead: launched a disposable Extension Development Host
(`/tmp/ahar-visualizer-demo-<epoch>`), confirmed its `devQueue` heartbeat was live, then
`traversal-compare view <run-id>` — `viewer.py` prefers the dev-queue channel and routed both
panels there automatically. Screenshot confirmed both trees rendered correctly.

Updated docs to make this the documented path rather than something rediscovered each time:
`ahar-visualizer-dev-workflow.md` (new section + broadened frontmatter description),
`references/REFERENCES.md`'s pointer, `traversal-compare/references/architecture.md`'s viewer
step description, and a new "step 0" in both `running-a-test/SKILL.md` and
`viewing-results/SKILL.md` — launch the dev-host and confirm the heartbeat before relying on live
visualization at all.

## Why with-harness lost on tokens/time

That run passed both variants (`end_accuracy: 1.0`) but with-harness cost more: 38,379 tokens /
123.1s vs. without-harness's 30,875 tokens / 80.1s. Traced it in the transcripts rather than
guessing. Finding: the `agent-harnesses` metaskill was never invoked, on either variant, across
all 6 steps — zero `Skill` tool calls anywhere in the cumulative with-harness transcript. Expected,
per `efficient-exploration`'s own design (`SUITES.md`: "no skill-invocation trigger phrase on
either side," testing organic discovery) — but the consequence is that with-harness's fixture,
which has extra `SERVICES.md`/`README.md` routing files sitting alongside real content at every
directory, only added cost this run: 34 tool calls vs. 25, including 4 routing-file `Read`s
(`SERVICES.md` ×3, `README.md` ×1) that without-harness's fixture doesn't even have. The model's
plain `grep -ril`/`find` kept surfacing those routing files alongside real targets, and it spent
extra turns opening them out of curiosity with no navigational payoff, since it was never actually
using the routing mechanism to skip anything — still finding everything by brute-force search
regardless. A real, legitimate result: routing files are dead weight unless something gets the
model to actually consult them.

## Harness priming: designed, not yet built

User's response: before a with-harness continuum starts, fire `"load harness"` as its own `claude
-p` call (only for the harness-enabled variant), tracked as a *separate* cost/time record that
never touches any step's own metrics — once per new chat continuum (a fresh, non-`resume_previous`
session start), not once per step, and not repeated when a continuum resumes.

Flagged before designing further: this is structurally close to the `prompt_prefix`
force-triggering mechanism `methodology.md` already documents as a rejected "real confound" (an
untriggered model just greps its way there and never touches routing, so a forced-trigger result
measures "routing plus forced invocation," not routing under its own steam). The new design
doesn't resolve that tension so much as choose the other side of it deliberately: it changes what
`efficient-exploration` measures, from "does the model discover the harness on its own" (this
run's actual finding: no) to "given deliberate priming — cost disclosed and separated out — does
per-step navigation get cheaper afterward." Both are legitimate questions; the point is not to
let the docs go silently self-contradictory about which one this suite is now answering.

Planned implementation (researched, not yet written — checkpointed here mid-task on a usage-limit
warning):
- `claude_runner.py`: a `run_harness_priming()` wrapper around the existing `run_step()` (prompt
  `"load harness"`, `enable_metaskill=True`, its own session).
- `cli.py`'s `run_cmd`: for a with-harness continuum start (`resume_previous: false`), fire priming
  first with the continuum's session id, then have the actual first step *resume* that session
  (`-r`) rather than starting fresh with `--session-id` — so priming and the step it primes share
  one continuum/transcript, and the live viewer panel shows both in sequence. Priming results
  collected in a separate `priming_records` list, never appended to `step_records`, so the existing
  `tokens`/`wall_clock_seconds` aggregation excludes them with no extra guarding needed. A new
  `priming_metrics` dict (same `Metrics` shape, computed via the existing `metrics.compute_metrics`)
  stored alongside each variant's real `metrics`, not merged into it.
- `results.py`: a `snapshot_priming_transcript()` sibling to `snapshot_transcript()` (same
  subagent-dir handling, distinct `<variant>-priming<N>.jsonl` filename) — `view_cmd` needs no
  change, since the last real step's snapshot already includes priming's turns (same session file).
- Docs needing updates once built: `references/methodology.md` (the load-bearing one — explain the
  new mechanism against the rejected `prompt_prefix` precedent), `references/architecture.md`,
  `references/task-schema.md`, `running-a-test/SKILL.md`, `viewing-results/SKILL.md`,
  `reporting-results/SKILL.md`'s example table (a `harness_priming` row, separate from the headline
  three).
- Tests: `tests/test_claude_runner.py` (argv shape for the priming call),
  `tests/test_results.py` (new snapshot function) — matching existing per-function unit-test
  style; `cli.py`'s `run_cmd` itself has no existing test coverage (shells out to real `claude`),
  consistent with today's pattern.
