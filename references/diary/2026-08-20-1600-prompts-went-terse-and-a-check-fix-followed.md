User noticed `real-repo-exploration/agentdev-snapshot`'s prompts read like a written spec, not like
how anyone actually talks to Claude — full clauses naming the exact file/function to trace,
"do not edit anything" repeated on every step. Asked to take inspiration from real transcripts of
this very session's own back-and-forth and rewrite the prompts terser and more assumptive of
context, anticipating the checks would need to get more robust to match. Rewrote all six steps
(e.g. "does ahar validate actually fail if a subdirectory's missing its SERVICES.md, or is that
just a warning?" instead of a multi-clause spec with "don't assume — trace the actual code"
repeated); "do not edit anything" now appears once, on step 1, the way a person would actually say
it. Added `answer_contains`'s `any_of` form to `grading.py` for the conceptual half of a check,
keeping a real identifier as a separate strict check alongside it — documented as a new principle
in `methodology.md` ("Real prompts aren't that specific") and `defining-a-test/SKILL.md`.

A live run after the redesign (`20260820T154943Z-981c1b79`) failed 5/30 checks, which turned out to
be two different real things, not one bug:

- **A genuinely wrong answer.** Step 2 asks what happens when `.leaf-detectors` assigns the same
  path to two different leaf types. `full-harness` confidently answered "no error, silent
  first-match-wins by line order" — wrong; the real `load_leaf_detectors()` (parser.py:36-42) raises
  `ParseError`. Both `metaskill-removed` and `harness-removed` got this right by reading the actual
  parser code. Left both of step 2's checks as-is — they correctly failed a wrong answer. Notable on
  its own: the full-harness/metaskill condition got this wrong on this run while both no-metaskill
  conditions got it right.
- **A real check-design gap — took three iterations to actually close.** The correct answers from
  `metaskill-removed`/`harness-removed` both failed a strict check requiring the literal string
  `seen_paths` — parser.py's internal tracking dict's variable name. Neither correct answer happened
  to quote it, since "does it just pick one?" doesn't invite naming internal variables the way
  "what's the exact prompt, word for word?" does. First fix: swap to `"is already claimed by leaf
  type"`, the literal error-message substring — verified offline against both captured answers, but
  a live re-run immediately failed a *different* correct answer that quoted parser.py's source
  verbatim, because that exact phrase straddles an f-string line break in the real source. Second
  fix: `"cannot also assign to"`, the message's trailing clause, single-line in the source — but two
  more live runs produced correct answers that paraphrased the mechanism in their own words instead
  of quoting the message at all, failing that too. Landed on `"load_leaf_detectors"` — the actual
  function name, which every one of six real correct answers across four separate live runs named,
  since locating it by name is unavoidable in any real trace regardless of how the rest gets
  phrased. Function/identifier names hold up under paraphrase in a way no wording of a *message*
  does — the project's own `defining-a-test/SKILL.md` guidance, learned here the hard way.
- **Left alone**: `full-harness` also missed the heartbeat-freshness check on step 5 (named
  `devQueue.ts` as the fallback but never explained why `viewer.py` checks its `.heartbeat` file's
  freshness before trusting it) — a genuine content gap in the answer, not a check problem.

Same discipline as every other fix this session: read the real captured answer text before touching
a check, and only touch the ones that were actually wrong to require what they required.
