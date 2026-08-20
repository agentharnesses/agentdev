---
description: Removed the efficient-exploration suite entirely (real content proved more indicative, per the user's direct instruction) and decoupled its tests from any real suite content so this can't break the same way again. Separately, traced why a live-visualized harness-removed panel showed no activity overlay — not a rendering bug, but a real sandbox escape: under bypassPermissions a model can guess a real absolute path this same machine also has live, unsandboxed copies of, and it did. Closed the Read/Glob/Grep vector with a home-directory --disallowedTools denylist, verified live; the equivalent Bash vector remains open and is documented as such.
date: 2026-08-20 08:21 CDT
git:
  agentdev: 4886940 (dirty — this entry's own changes not yet committed)
  traversal-compare: 8b2bac1 (dirty — suite removal, test decoupling, sandbox-escape fix, all doc updates, not yet committed)
---

## efficient-exploration removed outright

User's instruction, direct: "get rid of the efficient exploration repo type. Looking at real repos
is more indicative." Not a deprecation-in-place — `suites/efficient-exploration/` (both
`cross-subsystem-lookup` and `platform-lookup`, and their fixture directories) deleted entirely
(`git rm -r`).

The one design choice worth recording: `tests/test_grading.py` and `tests/test_tasks.py` had
grown real dependencies on `cross-subsystem-lookup`'s actual `task.yaml` content — loading it live,
asserting against its real check list, hand-crafting a `_CORRECT_ANSWER` string matching its
specific six questions. Rather than just repointing those tests at `real-repo-exploration` (moving
the same fragility, not removing it — the next suite rename/removal would break them again),
`test_grading.py` was rewritten to use a synthetic literal check list and throwaway `tmp_path`
directories, no `tasks.load_task` at all. `test_tasks.py`'s suite-discovery/ambiguity tests were
generalized the same way — real-suite ambiguity testing already had its own synthetic fixture
(`test_resolve_task_bare_suite_name_raises_when_ambiguous`), so the "is a real suite currently
multi-task" assertion was dropped rather than chased to whichever suite happens to have two tasks
this week.

Every doc reference updated: `README.md`, `methodology.md` (including the historical narrative
sections — kept the actual findings, since they're still true and still explain why current
mechanisms exist, but reworded pointers to the now-deleted suite as retired rather than live),
`task-schema.md`'s example, `defining-a-test/SKILL.md` (the real-repo git-snapshot pattern is now
*the* reference example, not a "second pattern" alongside the hand-built one), `reporting-results/SKILL.md`'s
example table (now real numbers from an actual `real-repo-exploration` run, 3 columns not 2), both
`SUITES.md` files.

## "No nav info for harness-removed" wasn't a rendering bug

Asked to check on the live-visualized run's screenshot, where the `harness-removed` panel showed
zero activity overlay while the other two panels lit up normally. Root cause, traced in the actual
transcript rather than assumed: the model, under `--permission-mode bypassPermissions` (no
filesystem jail, only a `cwd`), ran `find`, `grep`, `diff`, and three direct `Read` calls against
real absolute paths entirely outside its sandbox —
`/Users/danielwarfield/Documents/GitHub/metaskill/agent-harnesses/...` (a real, separate, currently
different-version standalone checkout of one of this project's vendored dependencies that happens
to exist on this machine) and this very repo's own live `references/architecture.md`/`methodology.md`
— the actual files this session has been editing all day, not sandboxed copies. `metrics.py`
correctly recorded these as touches; none matched the sandbox root prefix, so the visualizer
correctly had nothing to highlight. Confirmed via a `diff -rq` command the model itself ran between
the outside checkout and the live `vendor/metaskill/` — it genuinely investigated routing logic
*outside its sandbox entirely*, in a condition specifically meant to have none of that available.

This is a risk category a hand-built fixture can never create: a synthetic fixture's content is
never a real, guessable, currently-live path on the machine running the comparison. Using this
actual repo as its own fixture means an agent that infers or guesses a plausible real path can
sometimes reach genuinely live, unsandboxed content — a real confound for a "no-harness" condition
specifically, since it can accidentally leak exactly the routing information the condition is
supposed to withhold.

## Fix: closed for Read/Glob/Grep, left open for Bash — deliberately, not by oversight

Tested empirically before committing to an approach (per the user's choice, "deny-list reads
outside the sandbox root"): a literal single-leading-slash `--disallowedTools "Read(/path)"`
pattern did **not** block an absolute-path read under `bypassPermissions` — silently a no-op. A
`//`-prefixed (filesystem-root-absolute) glob pattern did: `Read(//Users/danielwarfield/**)`
blocked it outright, with the sandbox itself (confirmed always under the system temp dir, never
under `$HOME`) staying fully readable. `--disallowedTools "Read"` (disabling the whole tool)
*also* didn't fully close it — the model just used `Bash cat` instead, since `Read` and `Bash` are
separate tools with separate deny surfaces.

Implemented as `claude_runner._HOME_ESCAPE_DENYLIST`
(`{Read,Glob,Grep}(//<home-relative-to-root>/**)`, computed from `Path.home()` at runtime, not
hardcoded to this machine's username), added to every `_build_interactive_argv` call — every
variant, not conditional on `enable_metaskill`, since this is a general sandbox-integrity guard,
not part of what the harness/no-harness comparison itself measures. Verified live: re-ran
`harness-removed` alone after the fix.

**Deliberately not attempted**: closing the equivalent `Bash`-tool vector. `Bash`'s argument is an
unstructured shell string, not a path field a permission pattern can glob-match the way
`Read`/`Glob`/`Grep`'s structured argument can — a command-prefix deny like `Bash(cat *)` would
also block every legitimate in-sandbox use of the same commands, which are exactly what most
real exploration in this project actually uses. Tested whether the model's own judgment might
decline an out-of-scope read on its own initiative (a manual test where it refused to `cat` a
`.zshrc` file, citing secrets and suspecting a directive-injection test) — but that's a heuristic,
not a guarantee, and the original `harness-removed` run's `find`/`grep`/`diff` calls against the
escaped path show it doesn't reliably trigger when the target doesn't *look* like a secret.
Documented explicitly in `methodology.md` as a real, currently-open risk rather than papered over.
