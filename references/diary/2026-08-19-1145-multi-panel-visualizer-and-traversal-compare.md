---
description: Built ahar-visualizer's side-by-side multi-panel capability, then used it to bootstrap traversal-compare — a new submodule that sandboxes claude CLI sessions against paired with/without-harness fixtures to measure exploration efficiency and consistency.
date: 2026-08-19 11:45 CDT
git:
  agentdev: a1374bc (dirty — this entry's own changes not yet committed)
  ahar-visualizer: 7acfe35
  traversal-compare: 53e0c4a
---

## Why

The standard's two claimed benefits — more *efficient* agent exploration and more *consistent*
agent exploration (landing on canonical information rather than duplicates/decoys) — had no way
to be demonstrated concretely. Doing so needed two things `ahar-visualizer` couldn't do yet: watch
more than one `claude` session at once, and let something other than a human clicking through VS
Code decide which directory/session a panel watches.

## ahar-visualizer: independently-configured panels

`HarnessTreePanel` was a hard singleton tied to `workspaceFolders[0]`, and `TranscriptWatcher`
always auto-followed whichever transcript in that one directory's `~/.claude/projects/<slug>/`
was most recently modified — flagged as an open question on day one
(`2026-08-18-1101-ahar-vsvis-feature-plan.md`: "how the visit log and live visualizer disambiguate
between sessions if more than one is running").

Resolved by adding, alongside the untouched default singleton path: `HarnessTreePanel.createCustom({rootPath,
sessionFile?, label?})`, which always opens a new, independently-tracked panel (never
reused/revealed); `TranscriptWatcher`'s new `sessionFileOverride` param, which pins a panel to one
exact transcript file instead of auto-following, retrying every tick until it exists (so a panel
can be opened *before* the session that will populate it starts); a Command-Palette-only advanced
command for interactive use; and — the piece that matters for automation — a URI handler,
`vscode://agentharnesses.ahar-visualizer/openTree?root=...&session=...&label=...`, so an external
script can pop a pre-wired panel with zero VS Code UI interaction. Per-panel debug logs too (the
old single fixed-path log would have let two concurrent panels truncate/interleave each other's
trail). Full contract in `ahar-visualizer/references/multi-panel-testing.md`.

## traversal-compare: a new submodule

Bootstrapped a brand-new repo, `agentharnesses/traversal-compare`, as a comparison-test framework,
added top-level here (product repo, like `ahar-visualizer` — not `vendor/`, which is reserved for
copies of upstream standard/tooling repos). Scaffolded via `ahar init` (dogfooding the standard for
the framework's own structure) with a Python runner
(`src/traversal_compare/`): copies a fixture into an isolated sandbox, launches `claude -p
--session-id <uuid>` (the caller-chosen id makes the eventual transcript path computable *before*
running anything, so the URI handler above can be fired first with no discovery race), extracts
file-touch metrics from the transcript (a deliberate Python reimplementation of
`TranscriptWatcher.extractFilePaths`, not shared across languages — same choice `harness.ts`
already made relative to the metaskill's Python), grades deterministically against the task's
declared checks, and stores a `result.json` + transcript snapshots.

Fully worked one task end to end — `suites/efficient-exploration/add-currency` — real fixture
pair (an "orders service," ~18 files/variant, byte-identical decoys between variants so routing
signal is the only independent variable), a real completed run, committed as proof.
`suites/consistent-exploration` (the "duplicate canonical record" / deviation-across-sequential-steps
test) is scaffolded as `status: stub` only — deliberately deferred, not this pass.

Two real bugs found only by actually running it, not by review:

- `metrics.py` double-counted files because `claude` reports symlink-resolved absolute paths
  (macOS's `~/Documents` can itself be a symlink) that didn't string-match a merely-joined sandbox
  root. Fixed by resolving both sides before comparing; regression test uses a real symlink.
- The with-harness fixture had routing files on disk but nothing actually invoking the
  `agent-harnesses` skill — a capable model just `grep`/`find`'s its way to a small fixture and
  never touches the routing at all, silently defeating the comparison. Fixed by vendoring the
  metaskill into the fixture (`sandbox.py` generates the matching `.claude/settings.json` plugin
  registration fresh per sandbox, since the path is sandbox-specific and can't be committed) and
  adding a `prompt_prefix: "load harness"` on the with-harness variant's first step — matching this
  project's own convention for triggering the skill. Confirmed with a before/after run: without the
  prefix, with-harness's `claude` call used raw `grep`/`find` exactly like without-harness did;
  with it, `Skill agent-harnesses` → `summarize.py` → `SKILL.md` → target, zero wasted touches.

## Where this leaves things

`ahar-visualizer`'s multi-panel work and `traversal-compare`'s bootstrap are both committed
locally; `traversal-compare` is pushed and registered as a submodule here. `ahar-visualizer`'s
commit is *not yet pushed* — the currently-installed `ahar-visualizer` VS Code extension predates
it, so `traversal-compare run`'s live side-by-side panels won't actually pop until it's
repackaged/reinstalled (or run via the `dev-preview` skill's Extension Development Host).

Next: author real fixture content for `consistent-exploration` — the schema, `resume_previous`
machinery, and stub are already in place for it.
