---
description: Corrected accounting for the swe-bench-pilot result -- once the answering session's harness-priming turn and harnessify's own one-time prep-session cost are both included (not just the answering step's own tokens), the token story is far more modest than first reported; the time story stays strongly positive.
date: 2026-08-21 17:20 CDT
git:
  agentdev: b3c37a0
  traversal-compare: 31b46f1
---

User caught a real gap right after `2026-08-21-swe-bench-pilot-a-real-5x-token-6x-time-win.md`:
"For harnessified we should also be recording the time/token cost of harnessify itself, and the
time/token cost of the initial load harness." Checking, the load-harness priming cost was already
recorded (`variants.<vid>.priming`, always has been) but never surfaced in that entry's or my own
ad hoc chat report's numbers. The harnessify prep session's own cost wasn't recorded anywhere at
all -- `materialize_harnessified_fixture` called `session.send(...)` twice and threw both
`StepResult`s away.

Fixed both in `traversal-compare` (`31b46f1`): `harnessify.py` now aggregates both prep-session
turns via `metrics.compute_metrics` and persists the result as a `<key>.cost.json` sidecar next to
the cached fixture (never inside it, so it can't leak into a sandbox's diff), readable via a new
`cost_for_fixture()` even on a cache hit; `cli.py` attaches it to a harnessify-sourced variant's
result record as `harnessify_cost`. `skills/reporting-results/SKILL.md` now instructs adding a
`harnessify_prep (.../one-time)` row alongside the existing `harness_priming` row in every cost
table, both disclosed but never folded into the headline per-run numbers. Live-verified against a
small real repo (40,976 tokens, 25.1s captured correctly; a cache hit still returns it in 0.01s
with no re-run).

The already-cached `psf/requests` fixture from the pilot predates this change, so it has no
sidecar -- but its real transcript still exists, so the actual number could be computed directly
rather than left as a gap:

| | baseline | harnessified |
|---|---:|---:|
| answering step | 89,589 tok / 427.4s | 17,833 tok / 69.2s |
| harness-priming (answering session) | — | 42,384 tok / 10.4s |
| **per-question total** | **89,589 tok / 427.4s** | **60,217 tok / 79.6s** |
| harnessify prep (one-time, amortized) | — | 302,207 tok / 209.4s |

**Corrected picture:** folding in priming, harnessified used **~1.5x fewer tokens** per question
(60,217 vs 89,589), not ~5x as first reported -- the earlier number quietly compared only the
answering step's own tokens. The **time** advantage holds up much better: **~5.4x less wall-clock**
per question (79.6s vs 427.4s) even with priming included, since baseline's expense was dominated
by slow, serial operations (interpreter installs, network calls, sleep/poll loops) that don't cost
much in tokens but cost a lot in time.

The one-time prep cost changes the shape of the finding entirely, in opposite directions for the
two metrics. It's a real, large cost -- 302,207 tokens (more than baseline's entire single-question
cost) and 209.4s. Amortized across `N` questions against the same harnessified repo, breakeven
against baseline is:
- **Tokens**: `N ≈ 302,207 / (89,589 − 60,217) ≈ 10.3` — needs roughly eleven questions against the
  same repo before harnessify's token cost is recovered.
- **Time**: `N ≈ 209.4 / (427.4 − 79.6) ≈ 0.6` — pays for itself in wall-clock terms before even
  finishing the *first* question, since baseline's slow verification detour alone (~350s) already
  exceeds the entire one-time prep session's wall-clock cost.

So: on this one real pilot instance, harnessify is a near-certain time win from question one, but a
token win only once a harnessified repo gets reused across roughly a dozen questions -- a
materially different (and more honest) claim than "5x fewer tokens." This is exactly the kind of
correction the recording gap made invisible; worth remembering for every future report:
**a harnessify-sourced variant's true cost is step + priming + prep-amortized-over-reuse-count,
never step alone.**
