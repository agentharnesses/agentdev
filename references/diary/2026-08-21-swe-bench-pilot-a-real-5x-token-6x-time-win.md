---
description: First real external-repo harnessify pilot (psf/requests, SWE-bench Lite psf__requests-2317) -- a striking positive result, not a wash, after every prior real-repo comparison on agentdev's own content came out flat.
date: 2026-08-21 16:30 CDT
git:
  agentdev: 0c3dce7
  traversal-compare: abe60c9
---

Ran `swe-bench-pilot/question-set-1` for real, both variants, right after building it. One real
external repo (`psf/requests`), one real GitHub issue (`psf__requests-2317` from SWE-bench Lite,
taken verbatim), pinned at its real `base_commit`. `baseline`: plain export, no routing, one
session just answers. `harnessified`: a separate one-time session runs `ahar init` + `agent-harnesses`
+ `harnessify` to author real routing first (cached), then a *fresh* session answers.

First attempt failed -- `materialize_harnessified_fixture`'s harness-priming turn timed out at
600s with no transcript ever created. A standalone diagnostic (same `InteractiveSession` call,
instrumented to poll instead of block) produced a transcript in 5.3s, so the mechanism itself
wasn't broken; the likely cause was resource contention from this very session's own heavy
foreground tool activity at the moment the background run started. Re-ran with no concurrent
foreground work -- succeeded cleanly.

Real result, both variants reached `end_accuracy: 1.0` (all three checks passed -- correct fix,
correct root-cause diagnosis, real verification described, on both):

| | baseline | harnessified |
|---|---|---|
| wall clock | 427.4s | 69.2s |
| total tokens | 89,589 | 17,833 |
| input tokens | 74,642 | 11,649 |
| output tokens | 14,947 | 6,184 |
| files touched | 6 | 3 |

Harnessified used **~5.0x fewer tokens** and **~6.2x less wall-clock time** for an equally correct,
equally well-verified answer. `baseline`'s transcript shows why: it burns most of its budget
grep-spelunking `compat.py`/`utils.py`/`sessions.py` to relocate `builtin_str`/`to_native_string`
and confirm which one is already imported, then goes further and stands up a whole Python
2014-era-compatible interpreter (Python 3.9 via `uv`, a `cgi` module shim) to actually run the real
test suite against live httpbin.org for verification. `harnessified`'s `HARNESS.md` (real,
agent-authored, not templated) pointed straight at `requests/REQUESTS.md` for "changing or
understanding library behavior" -- the session found the bug site almost immediately and verified
by isolating and running just the relevant conversion logic directly, skipping the full
interpreter-reconstruction detour entirely. Both approaches were legitimate and both fully
satisfied every check; the harnessified path was just far more direct.

This is the first real, external, positive result after `2026-08-21-1340-a-wash-even-on-consistency-and-cost-methodology-in-doubt-not-the-standard.md`
found cost a three-way wash and consistency signals disagreeing across three question sets, all on
`agentdev`'s own tidy, self-referential content. That entry's own hunch -- routing's value might be
real but hard to see when cold search is already near-optimal on a repo the agent (and the test
author) already half-knows -- gets real support here: `psf/requests` is genuinely unfamiliar
content, `baseline` had no choice but to explore it from scratch, and the gap showed up immediately,
on the very first real instance tried.

Standard caveats apply hard: **n=1**, a single instance, a single repeat, one small/tidy repo
(~13MB) picked partly *because* it was small and fast to iterate on -- not yet evidence this
generalizes to SWE-bench's larger repos (astropy, django, sympy) or holds up over repeats (a lucky
baseline exploration path, or an unlucky one, is entirely possible at n=1). The harnessify prep
session's own real cost is excluded from these numbers by design (amortized once per repo+ref,
cached across every future run against it) -- fair for a suite meant to be run repeatedly against
the same target, but worth stating plainly rather than letting the token table imply routing is
free. Natural next steps, not yet started: more repeats of this same instance to see if the gap
holds, then more instances (including at least one from SWE-bench's larger repos, to test whether
the effect holds or shrinks as `baseline`'s cold-search cost is spread across more surface area
either way).
