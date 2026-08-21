Asked to assess feasibility of deriving a new repo from Tencent-Hunyuan/RepoProbe (ASE 2026): a
benchmark of 500 real questions sourced from actual GitHub Discussions across 50 repos, answered by
a Docker-isolated agent (any model, any scaffolding, swappable via a small container contract —
repo at `/app/repo`, question on disk, answer via stdout), graded by a weighted, tiered checklist
an LLM judge scores item-by-item (e.g. "5 points: correctly explains X — 3 points: mentions X
without the mechanism — 0: wrong"), not a scalar rating. Apache-2.0 code, CC BY 4.0 data — forking
is legally unrestricted.

Verdict: literally forking the repo is low-value. RepoProbe's core abstraction is *hold the
repo+question fixed, swap the model/agent* — `traversal-compare`'s is the opposite, *hold the
model/agent (Claude Code) and repo fixed, swap whether Agent Harnesses routing is present*. Their
harness has no notion of "routing file presence" as a variable; retrofitting one means gutting
their Docker/gateway machinery and rebuilding the `--plugin-dir`/fixture-stripping control
`traversal-compare` already has — you'd re-derive `git_fixture.py`/`sandbox.py`/`cli.py`, not save
that work. What's actually worth taking: their weighted/tiered checklist grading design (a real,
incremental upgrade path for `grading.py`'s currently-binary `checks[].metric`, unrelated to
forking anything), and `repos_info.json`'s pinned-commit-per-repo shape, which matches
`git_fixture.py`'s existing pattern closely enough that materializing their repos through our own
pipeline would be low-friction if we ever wanted to.

Follow-up question sharpened the actual opportunity: what if we added real Agent Harnesses routing
to (a subset of) RepoProbe's 50 repos and reran their real, externally-sourced, already-checklisted
questions with/without it? This fixes the biggest validity gap in `real-repo-exploration` as it
exists today — we wrote the routing, the questions, *and* the checks for `agentdev`, its own single
self-referential repo, which risks the routing and questions unconsciously converging on each
other's shape. RepoProbe's questions come from real Discussions we had no hand in, on repos we
didn't write, checklisted against real human answers — a much harder-to-fool signal if routing
shows a real effect there too.

The concrete gap standing in the way, named directly: `fixture_source` in `question-set.yaml` today
can only *strip* routing (`strip_routing`/`strip_metaskill`) — there's no mechanism to *add* it to
an arbitrary external repo that doesn't have any. Authoring genuine, accurate routing for 50 real
repos across 15 languages by hand is real understanding-work per repo, not a script — a scoped
pilot (4-6 repos, weighted toward questions tagged "Project Architecture" where routing plausibly
helps most) was the recommended shape, not attempting all 50 at once.

User's response reframes this as a feature in its own right, not just test-harness plumbing: a
**"harnessify" skill for the metaskill itself** — given an arbitrary existing repo with no Agent
Harnesses routing at all, add the real constructs (`HARNESS.md`, nested `SKILLS.md`/`SERVICES.md`-
style routing files, `.leaf-detectors` where warranted) onto it, grounded in the repo's actual
structure rather than boilerplate. Two things at once: a real capability the standard doesn't
currently offer (every existing tool here — `ahar validate`, the metaskill's `disclose.py`, this
project's own fixture-authoring guidance — assumes routing already exists; nothing *creates* it),
and, not incidentally, exactly the missing mechanism that would let `real-repo-exploration` (or a
new suite) pilot-test routing's real impact against RepoProbe's external, unbiased repo set instead
of only ever `agentdev`. Not yet scoped or built — this entry exists to not lose the idea before
the next session picks it up. Natural next steps, whenever this gets picked up: decide whether
"harnessify" lives in `vendor/metaskill` (upstream, a new capability of the standard itself) or as
a project-local tool here that only this repo's evaluation pipeline uses; and whether a first pilot
targets `agentdev`'s own currently-unrouted vendored dependencies (`vendor/agentharnesses`,
`vendor/cli`) as a zero-external-repo dry run before touching anything from RepoProbe.
