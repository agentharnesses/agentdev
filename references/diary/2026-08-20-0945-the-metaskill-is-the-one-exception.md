---
description: User caught a real design flaw mid-investigation — the metaskill's own real source code was never stripped from metaskill-removed/harness-removed (only routing files were), letting an agent manually reconstruct routing logic the structured tool would have given it. Fixed by treating the metaskill as atomic infrastructure, stripped wholesale from both no-metaskill conditions, which forced replacing two of the task's six questions. Found and fixed two more Bash-path-extraction false positives (find -name/-iname, KEY=VALUE assignments) along the way, and corrected an earlier overly-pessimistic claim about the sandbox-escape fix leaving Bash fully open.
date: 2026-08-20 09:45 CDT
git:
  agentdev: 3df61f6 (dirty — this entry's own changes not yet committed)
  ahar-visualizer: 91ad3b9 (dirty — transcriptWatcher.ts/test changes not yet committed)
  traversal-compare: 4603941 (dirty — git_fixture.py, task.yaml, metrics.py, doc changes not yet committed)
---

## What the user caught

Mid-investigation into a sandbox-escape subagent dispatch, the user stopped to ask what it was
actually looking for — a subagent prompted to find "routing filename reset logic" and
".harnessleaf precedence" in a real external checkout of the metaskill. Their read: the fixture
design had a real gap. `metaskill-removed`/`harness-removed` only ever stripped *routing files*
(`HARNESS.md`, `.harnessleaf`, `.leaf-detectors`, propagated index files) — never the metaskill's
own real source (`vendor/metaskill/agent-harnesses/`, the `.claude/skills/agent-harnesses` symlink,
`traversal-compare`'s own independent copy). None of that is shaped like a routing file, so the
existing stripping logic correctly left it alone — but that meant an agent under "no metaskill"
could still find and read (or dispatch a subagent to read) the metaskill's actual implementation
and manually reconstruct exactly what the structured `Skill` tool would have surfaced. The
structured access was gone; the information wasn't.

User's resolution, stated directly: "the metaskill is an atomic thing that is outside the scope of
the consistency rule, because it doesn't actually contain any information about the repo; it's
just a skill the agent harnesses standard uses. If we remove the 'agent harnesses standard', then
it's fine. So, it should be removed for both of the removals."

## Implementation: strip_metaskill, independent of strip_routing

`git_fixture.strip_metaskill_dirs(src, dest)`: removes every `agent-harnesses` directory that's
actually a skill leaf (has its own `SKILL.md`) — a real directory, or a symlink resolving to
one — wherever it appears, wholesale. Two-pass, same discipline as `strip_routing_files`: collect
every match against the intact tree first, then delete, so removing one match can't corrupt
detection of another. `materialize_variant_fixture` gained a second, independent boolean
(`strip_metaskill`) composed with `strip_routing` as each variant needs — three real combinations:
neither (`full-harness`), metaskill only (`metaskill-removed` — routing files stay, the metaskill
alone goes), both (`harness-removed`). Verified against the real pinned snapshot: exactly three
`agent-harnesses` skill-leaf directories exist in the real repo, all three removed cleanly, nothing
else in the 529-file tree touched (confirmed by diffing full file lists), `ahar validate` still
clean.

## Consequence: two of six questions had to be replaced

Steps 1 and 2 asked about the metaskill's own internals (`find_top_level_dir_name`'s
nested-boundary reset, `.harnessleaf`/`.leaf-detectors` precedence in `disclose.py`) — genuinely
good questions when written, now unanswerable in two of three variants by design. Went looking for
real, non-metaskill content to replace them with and found something better than a like-for-like
swap: the real `ahar` CLI's own reference validator/parser
(`vendor/agentharnesses/harnesses-ref/`), imported by `vendor/cli`'s `main.py` as `validate_cmd` —
itself a small surprising fact (the CLI tool's real validation logic lives in the separate spec
repo, not in `vendor/cli` despite `ahar` being its own name). Reading that code turned up two more
genuinely surprising, previously-unknown-to-me real facts: a missing routing summary file is only
ever an advisory `warnings()`, never part of `validate()`'s own `errors` list, so `ahar validate`
still exits 0 despite it; and a `.leaf-detectors` file assigning the same path to two conflicting
leaf types raises a real `ParseError` rather than silently picking either one. Both verified against
the actual pinned commit's code before writing checks, same discipline as every other question in
this task. New step 1 and step 2 trace these.

## Two more Bash-extraction false positives, found by the same live-verification discipline

Re-running the redesigned task live surfaced two more gaps in yesterday's Bash-path-extraction fix
(same session, same day — this whole investigative arc happened continuously):

1. **`find -name`/`-iname`** — `find / -name "claude_runner.py"` uses the filename as an
   exact-match *search pattern*, not a path to read, but it's syntactically indistinguishable from
   a real path argument. Extracted and resolved against `cwd`, it produced a plausible-looking but
   wrong, nonexistent path (`<sandbox_root>/claude_runner.py` instead of the real, nested file),
   silently displacing a real touch slot — worse than a miss. Fixed the same way a redirect target
   is: the token after a bare `-name`/`-iname` is never a path candidate.
2. **Bare `KEY=VALUE` assignments** — a live transcript had the model heredoc-writing a test
   `.leaf-detectors` file whose own *content* included lines like `skill=SKILL.md` and
   `mcp-server=SKILL.md` — real `.leaf-detectors` syntax, not file paths, but the value half has a
   recognized extension, so the whole token still passed the path heuristic. Fixed by excluding any
   token containing a literal `=` — a genuine file path essentially never contains one.

Both fixed in `metrics.py` and `transcriptWatcher.ts` in lockstep, both re-verified against the
real transcripts that surfaced them (re-running `compute_metrics` offline against the same captured
data, confirming the bogus entries disappear), both covered by new regression tests in each
language (91/91 Python, 113/113 TypeScript passing).

## Correction to yesterday's sandbox-escape write-up

While chasing the subagent-dispatch question, found that a follow-up live test contradicted the
original, more pessimistic claim that the `Bash` vector was left fully open by the
`--disallowedTools` fix. Reproduced deliberately: `find`, `ls`, `cat`, `grep` run via `Bash` against
a `$HOME`-denied path were refused with a clean system-level `Permission to use Bash with command
... has been denied.` message, 3/3, using the exact same deny pattern this project's own
`_HOME_ESCAPE_DENYLIST` sets. Claude Code's own permission engine appears to extract path-like
arguments from `Bash` commands and cross-check them against active `Read`/`Glob`/`Grep` path-scoped
deny rules too — more effective than originally documented. Still unverified: whether a *dispatched
subagent* inherits the same restriction (the investigation into this was still in progress, on the
exact case that prompted the user's "what's it trying to find?" question, when the metaskill
redesign superseded the need to finish chasing it — the new questions don't require the metaskill's
content at all, so even a successful escape there wouldn't help answer them anymore).

## Where this leaves things

Live-verified end to end after all of the above: 30/30 checks passed, all three variants at 100%
end-accuracy, `metaskill-removed`/`harness-removed`'s `distinct_files_touched` clean of both
metaskill-skill-dir paths and both new false-positive classes. `git_fixture.py`, `metrics.py`,
`transcriptWatcher.ts`, the task's questions/checks, and every doc reference — all built, tested,
and verified live — not yet committed as of this entry.
