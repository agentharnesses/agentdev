---
description: Design session (no implementation yet, no tokens available to run it) for a new
  swe-bench-pilot experiment -- does a harnessify-authored harness get MORE useful as it's carried
  across a sequence of real, chronologically-ordered codebase states and refined by the agent's own
  introspection, rather than being re-authored from scratch per instance as every prior batch has
  done. Landed on: introspection-only updating (no separate diff-based adaptation step), a
  low-drift Django 3.0 chain as the pilot target, and three new growth metrics (file count,
  character count, diff size) on the harness artifact itself.
date: 2026-08-27 14:27 CDT
git:
  agentdev: b44ae87 (dirty -- this entry not yet committed)
  traversal-compare: e57ba0d (dirty -- 2026-08-27 auto-archive/cli.py/skills work, not yet
    committed)
---

## The motivating gap

Every swe-bench-pilot batch so far (see `SUITES.md`) measures harnessify **cold**: one instance,
one fresh `harnessify.materialize_harnessified_fixture` call, one answer, thrown away. That's
correctly scoped for "does agent-authored routing beat no routing on a single real bug," which is
the question every batch to date has asked. It cannot answer a different, arguably more central
claim of the agent-harnesses standard itself: that routing files **improve with practical use** --
nothing in the existing pilot ever reuses a harness across more than one problem, so that claim has
never actually been tested here.

User's proposal, this session: chain together a sequence of real instances against the *same repo*,
ordered so each one represents a later state of that codebase than the last, and instead of
re-harnessifying from scratch at every step, carry the harness forward -- refining it based on the
agent's own reflection after each solve, not on whether the solve was graded correct. Measure
whether the harnessified-vs-baseline gap (cost, resolved-rate) widens, holds, or shrinks as the
chain progresses, and separately cost out however much the refinement step itself takes.

## Design, as it landed after discussion

**Chain selection.** Reused this same session's own version/temporal-density analysis of the full
(non-Lite) SWE-bench dataset: Django's 850 instances span 2015-08-19 to 2023-07-17 across 13
released versions, but a 30-day window (2019-03-30 -> 2019-04-29) contains 31 instances that are
**100% version-3.0-pure** -- the densest, least-structurally-drifted run of real issues found in
the dataset. This is the pilot's target chain. Ordering within it is by `created_at` as a
first-pass proxy for real commit ancestry -- a known approximation (two instances can share a
`base_commit`, or a later-filed issue can sit on an earlier commit) that would be worth replacing
with real git-history ordering against the already-cached Django clone
(`external_fixture.py`'s clone cache) before this is actually built, not just trusted as-is.

**Step 0 vs step N>0.** Step 0 is an ordinary `harnessify.materialize_harnessified_fixture` call,
exactly as every existing batch does it. From step 1 onward, the harness is **copied forward
as-is** onto the next instance's checkout -- no adaptation pass runs before the agent sees it.

**The update mechanism -- introspection only, diff-based adaptation dropped.** Original proposal
had two update triggers per hop: an introspection turn right after solving, and a separate
diff-based adaptation session (shown `git diff base_commit[N-1]..base_commit[N]`) before the next
instance starts. User's own reconsideration, mid-session: drop the diff-adapt step entirely. "The
harness will always be a little out of date; it's a lagging artifact. We're catching it up by
ordering things sequentially and asking for the update." This is a real simplification, not just a
scope cut -- with two update mechanisms in play, any observed improvement would be impossible to
attribute cleanly (introspection's own effect vs. the synthetic diff-nudge's effect); with only
introspection, the causal story is unambiguous. It's also a more honest simulation: no real harness
user diffs their codebase against the harness's last-known-state before every task -- they use it,
hit friction where it's stale, and fix it. Concretely, per step: solve (baseline + harnessified) ->
**same session**, one more turn -- "can the harness be improved to be more helpful?" -- **never
told whether the fix was graded correct** (grading is already a separate post-hoc Docker step per
`swebench_eval.py`, so blinding is just "never pass the grade into this prompt," not a sequencing
constraint) -> whatever harness results, edited or not, is what step N+1's checkout starts from.
User was explicit about why the blinding matters: "the updating isn't done based on the evaluation,
but on the LLM's own introspection, meaning we're not polluting it with a signal to the output" --
this keeps the experiment from quietly leaking the answer key into the harness. Same-session
(not a fresh transcript-reading session) was also a deliberate choice, user's own framing: "that's
the workflow, ask a question, get an answer, ask if the model can make the harness better."

**Scope of the pilot: low-drift only, for now.** A high-structural-drift contrast chain (crossing
Django version boundaries, e.g. 3.2->4.0->4.1) was floated as a natural companion experiment --
does introspection-only updating still work when the codebase is genuinely churning, not just
aging gently -- but user's call was to stay low-drift-only for the first real run. Worth revisiting
once introspection-only is validated here: if it doesn't hold up on the easiest case (minimal lag
between any two steps), a higher-drift chain isn't worth spending tokens on either.

**New growth metrics on the harness artifact itself.** User's addition, this message: instrument
how much the harness *itself* grows/changes at each step, not just the usual token/wall-clock/
resolved metrics on the solve. Three metrics, and the natural way to get all three essentially for
free -- store each meta-run's harness lineage as its own small git repo, one commit per step:

- **file count** -- `git ls-files | wc -l` (or equivalent) on the harness tree at each step's
  commit.
- **character count** -- total characters across every file in the tree at each step's commit.
- **diff size** -- `git diff --shortstat`/`--numstat` between two specific commits per step: the
  harness state the agent actually solved with, and the harness state immediately after that
  step's introspection edit (if any). This isolates *that step's own update footprint*, not
  cumulative drift since step 0 -- which is what actually answers "how big was this refinement,"
  and lets a later pass correlate update size against next-step improvement (does a bigger edit
  buy more, or does the harness converge to small, cheap tweaks over time).

This also solves resumability for free: a meta-run's harness history literally *is* its commit log
-- extending an existing meta-run is just adding more commits to that repo, and a sibling replicate
starting fresh is just a new repo with its own step-0 harnessify commit.

## What this needs that doesn't exist yet

- `harnessify.introspect_and_update(session, harness_dir)` -- the same-session follow-up turn,
  blind to the grade. Nothing like this exists; every current harnessify call is a one-shot,
  fire-and-forget authoring session with no return path.
- A meta-run store. `swebench_results_store.py`'s schema is a flat, unordered pool of independent
  instances (a batch's own `dataset.yaml` + `runs/<id>/result.json`, no concept of sequence or
  carried state) -- a meta-run is fundamentally an *ordered chain with state that carries forward*,
  which doesn't fit that shape at all. Needs its own store: an ordered list of steps, each with its
  own harness-repo commit ref, `harnessify_cost`/`introspection_cost` (new field, distinct from the
  existing one-time `harnessify_cost`) and the usual per-variant solve metrics.
- A real git-history-ordered instance sequence for the chain, replacing the `created_at` proxy.
- Whatever glue turns "the harness tree agent-harnesses-standard authors" into "a git repo with one
  commit per step" -- probably straightforward (it's already just files on disk), not yet written.

## Not yet done

Everything above is design only -- no code, no store schema, no pilot run. User has no tokens
available to run this right now; this entry exists so the design survives until there are. Suggested
first cut when resumed: N=5 steps, K=3 replicate meta-runs against the same fixed 31-instance
window (only step-0 harnessify's luck varies across replicates), before ever spending tokens on the
full 31-step chain or a high-drift contrast chain.
