---
description: User asked whether the model needs to regurgitate tool output during harness priming, since it's already in the transcript — traced the real cause to this project's own priming prompt combined with its verification-requiring system prompt, not the metaskill itself. Tightened HARNESS_PRIMING_PROMPT to discourage summarizing; live-verified a 3,277-output-token multi-paragraph recap collapsed to one sentence and wall-clock dropped from 42.6s to 12.3s. Also caught that the fixture is frozen at a pinned commit, so the task's own Q6 baseline had to keep describing the old prompt text, not the new live one. Found a second check-design flaw along the way — a genuinely correct answer that found a real bug in the vendored ahar CLI, failed only because a grading check expected a phrase split across a source-code line break.
date: 2026-08-20 10:32 CDT
git:
  agentdev: 3df61f6 (dirty — this entry's own changes not yet committed)
  traversal-compare: 4603941 (dirty — claude_runner.py, task.yaml changes not yet committed)
---

## The question

"If the summarize function is ran, does the model need to regurgitate the content of the harness?
Is the whole tool output within the chat continuum? Right now, I believe, the metaskill prompts
the model to summarize its findings."

Answered directly: no, it doesn't need to — any tool output (including `summarize.py`'s full tree)
is already part of the transcript, available to every later turn without being retyped. Checked
both `summarize.py` and the metaskill's own `SKILL.md` directly: neither instructs a summary.
Traced the real source to two of this project's *own* prompts instead —
`HARNESS_PRIMING_PROMPT` ("load harness — the harness root is your current working directory") and
`INTERACTIVE_PARITY_SYSTEM_PROMPT`'s "do not report something as loaded... unless you used a tool
to verify" — the model reads "prove you verified" as "write up what you found," and defaults to a
full prose recap. Confirmed with a real example: a priming turn had spent 3,277 output tokens
restating every submodule, subsystem, and diary entry, none of which anything downstream reads —
pure waste, since output tokens are the expensive kind.

## The fix

`HARNESS_PRIMING_PROMPT` gained a second sentence: "Once you've verified it's loaded, confirm
briefly; do not summarize what you found — the tool output is already in this conversation, so
restating it in your own words adds nothing." Deliberately targets the *behavior* (summarizing)
rather than the underlying instinct (prove you verified) — removing the verification requirement
itself would reopen the non-determinism `HARNESS_PRIMING_PROMPT`'s first sentence already exists to
prevent (see `2026-08-19`, "priming still didn't help").

Live-verified, not just reasoned about: a priming turn's output collapsed from 3,277 tokens across
several paragraphs to one sentence ("The harness at this working directory loaded successfully —
`HARNESS.md` found at the root, with nested harnesses under `ahar-visualizer/` and
`traversal-compare/`..."), and priming's wall clock dropped from 42.6s to 12.3s — roughly 3.5x
faster, on top of the token savings.

## A subtlety caught before it became a real bug

Went to update `task.yaml`'s baseline (step 6 asks the agent to quote the priming prompt "exactly,
word for word") to match the new prompt text — then realized this task's fixture is *frozen* at a
pinned commit (`4886940`), predating this edit entirely. The agent exploring the sandbox reads the
*old*, unedited `claude_runner.py` — confirmed directly against the materialized snapshot. The live
runner's own prompt and the sandbox content the agent traces are two genuinely different things
now, and that's correct, not a bug: reverted the baseline back to the old text, and added an
explicit note explaining why the two intentionally diverge, so a future maintainer doesn't "fix" it
back to matching the live prompt.

## A second check-design flaw, found by the same live-verification discipline

The full verification run failed one check — traced to a genuinely excellent answer, not a wrong
one. Asked to trace what happens when `.leaf-detectors` assigns the same path to two conflicting
leaf types, the model didn't just read the code — it wrote a real test fixture and ran the actual
`ahar validate` CLI end-to-end, and found a real, previously-undocumented bug in the vendored
`harnesses-ref` package: `validate_cmd` calls `validate()` (which correctly catches the
`ParseError`) and then unconditionally calls `warnings()` right after — but `warnings()`'s own
`load_leaf_detectors()` call has no `try/except` around it, so it re-raises the *same* `ParseError`
uncaught, crashing `ahar validate` with a raw Python traceback instead of a clean `✗ error` message.

The grading check (`answer_contains: "already claimed"`) failed only because the model quoted the
real source code's own multi-line f-string formatting verbatim — the literal string
`"is already "` and `"claimed by leaf type "` sit on two separate source lines, so the answer's
faithful reproduction of that layout broke the contiguous substring my check expected. Same class
of issue as `platform-lookup`'s Q6 months ago: a check too brittle against a correct answer, not a
correctness problem. Fixed by switching to `"seen_paths"` (the real variable name driving the
actual logic, guaranteed not to be split across a line break) — verified directly against the real
captured answer text before re-running, same discipline as every other check fix this project has
made.

## Where this leaves things

`claude_runner.HARNESS_PRIMING_PROMPT` tightened and live-verified; `task.yaml`'s step 2 check
fixed and offline-verified against the real answer that surfaced the issue; the baseline correctly
documents the deliberate live-prompt-vs-pinned-sandbox divergence. Full 3-variant live
re-verification in progress as of this entry.
