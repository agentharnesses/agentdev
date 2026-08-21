User asked for two things: a second real-repo-exploration question set whose first question is a
complex planning/design question asked cold (no earlier steps to warm the session up first), and —
tentatively — renaming "task" to "question set" throughout, since a suite can hold several. Asked
to confirm scope before touching anything this wide: chose the full rename (code, not just docs)
and chose to rename the existing nine-question task too, rather than leave it as a legacy name.

**The rename.** `tasks.py` → `question_sets.py` (`Task`→`QuestionSet`, `TaskError`→`QuestionSetError`,
`load_task`→`load_question_set`, `resolve_task`→`resolve_question_set`,
`discover_tasks`→`discover_question_sets`, `task_dir`→`question_set_dir`); `cli.py`'s
`task_ref`→`question_set_ref`, `grade_task`→`grade_question_set`, `"task_id"`→`"question_set_id"` in
`result.json`; every `task.yaml`→`question-set.yaml`; the old task directory
`agentdev-snapshot`→`question-set-1`; `references/task-schema.md`→`question-set-schema.md`; every
doc updated (`README.md`, `HARNESS.md`, both `SUITES.md`s, all four `skills/*/SKILL.md` files,
`references/{architecture,methodology,REFERENCES}.md`). Deliberately left the skill *directory*
names (`defining-a-test`, `running-a-test`) alone — "test" is a broader, still-accurate word for
what those skills cover, and the ask was specifically about "task," not "test." Verified with the
full test suite (94→95 passing after the rename, one expected failure until question-set-2 existed)
and a CLI smoke test (`list`), not just by reading the diff.

**The new question set — and the mistake it immediately caught, twice.** Designed a 3-step
`question-set-2`: a cold, complex planning question first ("I want to bump the pinned commit this
whole suite tests against — walk me through what that involves and what could quietly break"),
with two resumed follow-ups drilling into risk-prioritization and first-action. Contrast with
`question-set-1`, where synthesis questions come last, after six narrower lookups have already
built up session context — this one tests upfront design reasoning cold instead.

First live run: the whole premise broke the same way `question-set-1`'s steps 8-9 already had —
`git_fixture.py` (the actual mechanism the question was designed around) doesn't exist inside the
pinned snapshot at all, since the snapshot is frozen at a commit that predates it. But instead of a
shallow miss, all three variants independently reasoned from what *was* actually traceable and
landed on three completely different, genuinely valid framings: the meta-repo's own commit vs. its
submodules' independent pins; a real, previously-undocumented finding that the vendored metaskill
copy has zero recorded provenance (`ahar init`'s claude preset does a plain clone-copy-discard, no
SHA ever recorded anywhere); and the general "fixtures/checks tied to today's behavior can silently
stop reflecting tomorrow's" risk. A second live run added a *fourth* framing (reasoning from the
older hand-built `efficient-exploration` pattern instead) and broke a first-round check-fix that had
over-fit to one specific vocabulary ("before touching" vs. a differently-phrased-but-equally-correct
"before replacing"). Landed on two words, verified against all 6 real answers collected across both
runs before locking in: `silently`/`quietly` for the crux risk, `diff`/`Diff` for the recommended
first action — both present in every real answer regardless of which of the four framings it took.

Rewrote the question set's own `description` to describe what was actually found rather than assert
one intended mechanism, since the open-endedness turned out to be the interesting result, not a
flaw to hide. Both question sets confirmed passing live post-rename
(`question-set-1`: 20260820T201919Z-0acd9c61 PASS 45/45; `question-set-2`:
20260820T201355Z-dc7b8309 PASS 6/6).
