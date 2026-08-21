User asked why `real-repo-exploration` (two question sets, each in its own subdirectory holding a
single `question-set.yaml`) and `consistent-exploration` (one question set, its yaml sitting
directly at the suite root) used different directory shapes — and pointed out the real question
underneath: why does a question set need a whole directory at all when all it holds is one file?

The honest answer was history, not design: `consistent-exploration`'s shape predates multi-question-set
suites — a suite always had exactly one question set, at its root, alongside its `fixtures/`. When
`real-repo-exploration` needed two, `resolve_question_set`'s existing root-or-one-level-nested
fallback just got reused rather than actually unified, so multi-question-set suites got a
directory-per-question-set instead of a real redesign. And the reason each of those directories held
only one file: git-snapshot-sourced question sets never need a `fixtures/` tree at all (materialized
into a cache elsewhere by `git_fixture.py`), so the directory only existed because the old
one-fixture-per-question-set model required one.

Restructured every suite to the same shape, uniformly: `suites/<suite>/question-sets/<id>.yaml` —
one flat file per question set, never a directory just to hold one — plus an optional
`suites/<suite>/fixtures/<id>/{with-harness,without-harness}/`, namespaced by question-set id, for
a question set whose variants declare `fixture_path`. `question_sets.py`'s `QuestionSet.suite_dir`
field replaced `question_set_dir` (fixture/rubric paths now resolve against the suite directory,
since a flat yaml file has no directory of its own); `load_question_set` now takes the yaml file
path directly rather than a containing directory; `discover_question_sets`/`resolve_question_set`
rewritten for the flat layout. Moved `agentdev-snapshot`'s already-renamed `question-set-1`/
`question-set-2` directories into `question-sets/*.yaml`, and `consistent-exploration`'s
`question-set.yaml` + `fixtures/{with-harness,without-harness}/` into the same shape
(`fixtures/update-tom-opinion/{with-harness,without-harness}/`). Updated every doc that described
the old shape (`question-set-schema.md` gained a real "One suite, one shape" section with the
directory tree spelled out; `architecture.md`, `defining-a-test/SKILL.md`, both `SUITES.md` files,
`git_fixture.py`'s own docstring). 96/96 tests pass, `list` shows both suites correctly, and a live
run against the new layout confirmed the full pipeline — resolution, sandbox creation, grading —
still works end to end (`20260820T204044Z-71dbeba2`, PASS).
