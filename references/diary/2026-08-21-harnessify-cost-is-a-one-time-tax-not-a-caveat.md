---
description: Reframes 2026-08-21-full-accounting's "breakeven at ~11 questions" framing -- amends it, doesn't retract the numbers.
date: 2026-08-21 17:55 CDT
git:
  agentdev: 854c433
---

User correction on `2026-08-21-full-accounting-harnessify-prep-cost-changes-the-picture.md`: its
framing treated the one-time prep cost as a caveat that "changes the story" — as if needing ~11
questions to break even on tokens was a disappointing qualifier on the 5x-token headline. That's
backwards. The prep cost is *expected* to be significant — surveying and authoring real routing for
a whole repo is real, substantial work, done once. Nobody should be surprised it costs more than a
single question does. The number worth caring about was never "does it break even," it's the
**marginal per-query cost once the harness exists** — because a harnessified repo is meant to be
reused across many queries, and that marginal saving is what compounds with every one of them.

The real numbers from that entry stand unchanged (~1.5x fewer tokens, ~5.4x less wall-clock per
question once priming's included; 302,207 tokens / 209.4s for the one-time prep) — only the
framing was off. Restated correctly: harnessify pays a known, one-time, expected tax, and in
exchange every subsequent question against that repo runs measurably cheaper and much faster. The
"breakeven" framing implied the cost was in question; it never was — the repo either gets reused
enough to be worth harnessifying or it doesn't, and that's a judgment call about the target, not a
weakness the pilot uncovered.
