---
name: agentdev
description: Development harness for tooling built around the agent-harnesses standard — holds the ahar-visualizer VS Code extension and the traversal-compare comparison-test framework as submodules, plus shared developer knowledge.
---

## Upon loading the Harness

This meta-repo was originally scaffolded around a product called toprope (an Electron agentic client + companion CLI, held as git submodules). That direction was dropped — see `references/diary/2026-08-18-1007-vs-code-base-or-extension.md` for the full reasoning. It now holds two product submodules: `ahar-visualizer`, a VS Code extension that visualizes agent navigation and the agent-harnesses standard structure for repos that use it (including opening multiple independently-configured panels side by side — see its own `references/multi-panel-testing.md`); and `traversal-compare`, a Python framework that sandboxes `claude` CLI sessions against paired with/without-harness fixture repos to measure exploration efficiency and consistency, driving `ahar-visualizer`'s side-by-side panels live via its URI handler. See `references/diary/2026-08-19-1145-multi-panel-visualizer-and-traversal-compare.md` for how these two fit together.

This repo holds the shared `agentdev` harness itself (agent-harness-standard conventions and integration knowledge) plus those two product submodules. This repo was renamed from `toprope-agentdev` to `agentdev` on 2026-08-18 to drop the stale `toprope` product branding — see `references/diary/2026-08-18-2345-toprope-agentdev-renamed-to-agentdev.md`.

## Operating Notes

- **Do not close VS Code while a Claude Code session is running inside it.** The `claude`
  CLI runs as a normal child process of the VS Code integrated terminal — closing the
  window (or the terminal pane) kills that process immediately, with no chance to persist
  state or write a diary entry. This has already happened once and cost real time to
  recover from (see
  `references/diary/2026-08-18-1130-vscode-close-killed-session-mvp-recovered.md`). If a
  session needs to end, let it finish its current step and commit first, or explicitly ask
  it to wrap up — don't just close the window.

## How to Find Information for Claude

Use the `agent-harnesses` skill to explore the harness just in time, based on prompts from the user. Select only what is relevant and repeat until the session is complete, then read the returned resources.

When **maintaining the harness** (adding, moving, or renaming files), consult the `agent-harnesses` skill for reverse progressive disclosure to keep routing files in sync.

## Vendored Upstream (`vendor/`)

Three repos from the `agentharnesses` GitHub org are tracked here as git submodules, so local edits actually propagate upstream instead of drifting out of sync in a hand-maintained copy (see `references/diary/2026-08-18-1523-routing-filename-bug.md` for the incident that prompted this — this repo's own hand-ported copy of the metaskill had silently diverged from upstream with a real classification bug):

- `vendor/metaskill` — source of the `agent-harnesses` skill. **`.claude/skills/agent-harnesses` is a symlink into `vendor/metaskill/agent-harnesses`, not a real directory** — the skill's actual content lives in the submodule; don't recreate it as a plain directory.
- `vendor/agentharnesses` — the Agent Harnesses standard itself (`agentharnesses/agentharnesses`). **Its `docs/` are the ground truth for the standard** — when this repo's own explanations (skill docs, `harness.ts` in `ahar-visualizer`, etc.) disagree with it, the docs win; fix the local side, not the other way around.
- `vendor/cli` — source for the `ahar` CLI (`agentharnesses/cli`), installed locally (`ahar --help`). `ahar validate <path>` / `ahar show <path>` are a second, independent authoritative check — useful for catching cases where this repo's own tooling has quietly drifted from the standard, like the routing-filename bug above.

As of the commit that added them, none of the three have been modified from upstream — only referenced. Whether/how to contribute fixes upstream (e.g. the routing-filename bug) is still an open discussion, not yet acted on.

## Skills

- `skills/maintenance/` — harness upkeep, for this repo's own structure.
- See `skills/SKILLS.md` for the full index.

## References

- `references/diary/` — date/timestamped design diary for the toprope family of products.
- See `references/REFERENCES.md` for the full index.
