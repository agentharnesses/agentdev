---
description: Renamed the ahar-vsvis VS Code extension (and its checkout directory in this
  meta-repo) to ahar-visualizer — "vs" was redundant once the tool only ever runs inside VS
  Code. Local-only rename (directory, package.json, aharVsvis.* command/config IDs, current
  docs); the GitHub remote stays at agentharnesses/ahar-vsvis.git for now, deliberately.
date: 2026-08-18 16:29 CDT
git:
  toprope-agentdev: d3ef67e (dirty — this entry's own changes not yet committed)
  ahar-visualizer: c1e7bd4 (dirty — rename not yet committed)
---

## What changed

The extension was named `ahar-vsvis` (agent-harnesses + VS Code visualizer). Renamed to
`ahar-visualizer` — the "vs" was redundant: the tool only ever runs inside VS Code itself, so
qualifying it as "VS-something" adds nothing a user reading the name inside VS Code doesn't
already know.

Scope, decided explicitly rather than assumed: **local-only**. The submodule's remote is
`github.com/agentharnesses/ahar-vsvis.git` — a shared org repo, not just a local folder name —
so renaming the actual GitHub repo was treated as a separate, bigger, more-visible decision and
deferred. Only local references were renamed:

- The submodule's checkout directory: `ahar-vsvis/` → `ahar-visualizer/` (via `git mv` at the
  `toprope-agentdev` superproject level, which updated `.gitmodules`' `path` automatically; the
  submodule's logical name and `url` in `.gitmodules` were deliberately left as `ahar-vsvis` /
  `agentharnesses/ahar-vsvis.git`, matching the still-unrenamed remote).
- `package.json`: `name`, `displayName`, `contributes.commands[].command` (`aharVsvis.*` →
  `aharVisualizer.*`), `contributes.commands[].category`, `contributes.configuration.title`,
  and all four `aharVsvis.*` setting keys → `aharVisualizer.*`.
- `src/extension.ts` / `src/treePanel.ts`: matching command IDs, the
  `vscode.workspace.getConfiguration('...')` key, the webview panel view type, the debug-log
  path (`/tmp/ahar-vsvis-debug.log` → `/tmp/ahar-visualizer-debug.log`), and the settings
  button's tooltip text.
- `ahar-visualizer/README.md`, and this meta-repo's own `HARNESS.md` / `README.md` (the
  currently-authoritative, forward-looking docs).

**Deliberately not touched:** `references/diary/*.md` and `references/diary/REFERENCES.md`'s
one-line summaries of them. Diary entries are a point-in-time record of what was true when they
were written — several call the extension `ahar-vsvis` because that was its name at the time,
including one that's specifically *about* it being named that (the toprope → ahar-vsvis rebrand
entry). Rewriting history there would lose that record for no benefit.

## Verification

`tsc` compiles clean and all 53 `node --test` tests pass from the new directory location
(`npm install && npm test` inside `ahar-visualizer/`, confirming `package-lock.json`'s `name`
field picked up the rename too). Grepped the whole meta-repo (excluding `vendor/` and
`references/diary/`) for `ahar-vsvis`/`aharVsvis` afterward — nothing left outside the
deliberately-preserved diary history.

## Open follow-ups, not done here

- The `nested harness` / top-level-directory-reset work (added across `vendor/agentharnesses`,
  `vendor/cli`, `vendor/metaskill`, and confirmed in `ahar-visualizer/src/harness.ts`) and the
  auto-collapse quality-of-life settings added to `ahar-visualizer` (max depth, max nodes,
  max children, max nodes-on-expand, collapse-to-depth, expand-one-layer) both happened earlier
  in this session and don't have diary entries yet — this entry only covers the rename.
- Whether/when to rename the actual `agentharnesses/ahar-vsvis` GitHub repo (via
  `gh repo rename`) is still open, deferred per the local-only decision above.
