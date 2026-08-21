---
description: Built and live-verified the harnessify mechanism (minimal ahar init, a harnessify skill, external/harnessify fixture_source types) across three repos, catching two real bugs along the way.
date: 2026-08-21 21:10 EDT
git:
  agentdev: fbd0080 (dirty — submodule pointer bumps + this entry not yet committed)
  traversal-compare: 00f79a4
  vendor/cli: f9d3008
  vendor/metaskill: 2c2375a
---

Built and live-verified the "harnessify" mechanism sketched two entries ago
(2026-08-20-2334-repoprobe-assessed-and-a-harnessify-skill-idea-surfaced.md), across three repos in
dependency order.

`vendor/cli`: `ahar init` is now minimal by default — writes only `HARNESS.md`, nothing else. The
old preset-prompt flow (`.gitignore`, `.leaf-detectors`, a forced metaskill install) is now a single
opt-in `--metaskill` flag. When passed, `_install_metaskill` clones the metaskill repo once and
installs *both* `agent-harnesses` and the new `harnessify` skill from that one clone, and merges
into an existing `.claude/settings.json` (via `extraKnownMarketplaces`/`enabledPlugins`) rather than
overwriting it — necessary because a real external repo being harnessified will often already have
its own `.claude/` configuration.

`vendor/metaskill`: added a sibling skill directory, `harnessify/SKILL.md`, alongside
`agent-harnesses/`. Its instructions: survey the repo for real structure before deciding anything,
pick genuine leaf-type boundaries rather than defaulting every directory to `skill=SKILL.md`,
author routing grounded in what's actually there (never templated), and run `ahar validate .`
before finishing. `HARNESS.md` and `README.md` updated to describe two skills instead of one, with
an explicit warning that `agent-harnesses/` must stay at repo root since `ahar init --metaskill`
hardcodes that path.

`traversal-compare`: two new fixture_source types. `external_fixture.py` materializes an arbitrary
external repo (git URL or local path) at a pinned ref, reusing `git_fixture.export_tree` directly
(already generic) rather than reimplementing tree export — same content-addressed, outside-the-repo
cache pattern as every other fixture cache here. `harnessify.py` builds on it: resolve the ref,
check a second cache keyed by (source, resolved sha, a hash of the harnessify skill's own content),
and on a miss, copy the plain export into a working dir, run `ahar init <name>`, then drive a real
`claude_runner.InteractiveSession` through harness-priming + the harnessify prompt (needing both
`agent-harnesses` and `harnessify` loaded at once — this is what pushed `InteractiveSession`'s
single hardcoded metaskill `--plugin-dir` into a general `extra_plugin_dirs` list), then defensively
re-run `ahar validate` before caching the result. This is a genuinely separate, one-time session
from the later question-answering session — matches the user's explicit workflow: ahar init / load
harness / harnessify / new session / load harness / answer question.

Caught two real bugs by actually running this live against a small throwaway repo (`mylib`, a tiny
plaintext-to-HTML-AST library, no existing routing) rather than trusting the unit tests alone. One,
`InteractiveSession`'s `--setting-sources ""` (used everywhere in this project for reproducibility)
blocks project-scoped `.claude/skills/` discovery entirely, so a target repo's own on-disk
`harnessify` skill would never be found by the plugin loader — fixed by vendoring a real committed
copy into `traversal-compare/.claude/skills/harnessify/`, matching the existing `agent-harnesses`
copy, and pointing `extra_plugin_dirs` at it explicitly rather than relying on disk discovery. Two,
`_run_ahar_init` called `ahar init` with no name argument, so the generated `HARNESS.md`'s `name:`
field silently picked up the internal `.work-<key>-<pid>` cache directory's own basename instead of
anything about the repo being harnessified — only visible by actually reading the output file, not
from any test. Fixed by deriving a real name from `source` (last path segment, `.git` suffix
stripped) and passing it through; re-verified directly against a fresh copy of the same scratch
repo without re-running the full (slow) live claude session, since the fix is small, isolated, and
the surrounding pipeline was already proven live.

Aside from those two, the live run itself was a real positive result on its own terms: given only a
bare `HARNESS.md` and no other structure, the session correctly judged `mylib`'s three real
submodules as plain groups (no `skill=` leaf type forced on any of them), wrote grounded
descriptions matching what was actually in each file, correctly inherited the routing-filename
propagation convention for nested directories, and passed `ahar validate` on the first real attempt
— not templated boilerplate. 175 traversal-compare tests and 19 vendor/cli tests green throughout.
Not yet done, and deliberately out of scope for this pass per the approved plan: picking a real
external target (e.g. one RepoProbe or SWE-bench repo) and authoring an actual pilot question set
against it with the new `external`/`harnessify` fixture_source types.
