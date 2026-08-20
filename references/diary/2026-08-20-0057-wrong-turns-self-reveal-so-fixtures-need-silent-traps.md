---
description: Traced platform-lookup's cost breakdown (with-harness wins on real work, then gives most of it back to priming's fixed tax) and, prompted by the user, identified a deeper fixture-design gap — every current decoy self-reveals as wrong during exploration, so blind search barely pays for its wrong turns. The failure mode routing is actually meant to prevent (confidently landing on a wrong answer that never announces itself) isn't tested yet.
date: 2026-08-20 00:57 CDT
git:
  agentdev: 4886940 (dirty — pre-existing unrelated changes: references/REFERENCES.md, references/ahar-visualizer-dev-workflow.md, vendor/agentharnesses, vendor/cli, an untracked readme2.txt; none from this entry's work)
  traversal-compare: 8b2bac1
---

## Why with-harness didn't come out ahead on platform-lookup

User's expectation going in: the harness should make things faster. It didn't, net. Broke down
the `20260820T052156Z-3bba8493` run (8 questions, 3 deliberate module-naming traps — `identity/`
vs `authn/`, `notify/` vs `comms/`, live pricing vs a legacy decoy):

- with-harness's *actual task work*: 85.5s / 14,555 tokens — genuinely cheaper than
  without-harness's 84.5s / 26,127 tokens, and it never opened a single wrong decoy across all
  three traps (without-harness opened all three before landing on the right answer each time).
- But priming — the mandatory "load harness" turn fired before the continuum starts — adds
  another 17.8s / 15,588 tokens on top, pushing with-harness's real total to ~103s. That's a fixed
  cost paid once per continuum regardless of how many questions follow, and because priming shares
  the continuum's session, every downstream turn also carries that loaded context forward — the
  tax isn't isolated to the priming step itself.

On an 8-question task, the fixed tax isn't amortized away by the variable savings. Consistent with
what `2026-08-19-2032` already flagged as a live risk when priming was still just designed: it
trades "does the model discover the harness organically" for "given a disclosed, deliberate
priming cost, does per-step navigation get cheaper afterward" — and on this task, the answer to
the second question is "yes, but not by enough to pay for itself yet."

## Why blind exploration is so competitive: wrong turns self-reveal

The other half of the answer, independent of priming: guessing wrong isn't actually expensive here.
When without-harness opened `services/identity/` instead of `authn/`, or
`legacy/old_pricing_engine.py` instead of the live config, the wrong file told it so in one read —
no callers, an unrelated docstring, "deprecated" sitting right there in a comment or a filename.
Ruling out a bad guess costs one cheap turn, not a rabbit hole. With ~87 files total, brute-force
search converges almost as fast as routed search, because nothing in this fixture is willing to
lie convincingly.

## The gap this exposes (user's hypothesis, not yet built)

Prompted by the user: the fixtures may be too simple in a specific way — not too *small*, but too
*honest*. Every trap so far is a decoy that announces its own wrongness on inspection (no callers,
a deprecated marker, an empty implementation). That tests "can the agent recover from a wrong
turn," which blind grep is already good at. It doesn't test the failure mode routing actually
exists to prevent: an agent that finds something *plausible*, stops, and reports it confidently —
never triggering the self-correction, because nothing in the wrong file ever told it to look
further.

That requires a genuinely silent trap: a red herring that passes every shallow check a quick read
would apply — real callers, no deprecation marker, syntactically and semantically plausible output
— and is wrong only in a way that isn't locally visible from the wrong file itself, only from
context routing would have supplied up front (a `SERVICES.md` note like "this looks right but
X actually owns Y because Z"). Not yet designed or built; the honest next step is a task where an
agent can converge confidently on a wrong answer and *pass its own internal consistency check*
while doing so — at which point routing's disambiguation notes would be the only thing standing
between a plausible answer and a correct one, rather than a shortcut past a search that would have
self-corrected anyway.
