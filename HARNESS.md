---
name: toprope-agentdev
description: Development harness for tooling built around the agent-harnesses standard. Currently holds only shared developer knowledge (no product submodules) — see references/diary/ for how this repo's scope changed from an Electron app + CLI to a planned VS Code extension repo (ahar-vsvis, not yet added here).
---

## Upon loading the Harness

This meta-repo was originally scaffolded around a product called toprope (an Electron agentic client + companion CLI, held as git submodules). That direction was dropped — see `references/diary/2026-08-18-1007-vs-code-base-or-extension.md` for the full reasoning. The plan going forward is a single lightweight VS Code extension, `ahar-vsvis`, that visualizes agent navigation and the agent-harnesses standard structure for repos that use it; it's being developed as its own GitHub repo for now and may be folded in here later as a submodule.

Until that submodule is added, this repo holds only the shared `agentdev` harness itself: agent-harness-standard conventions and integration knowledge, not product source. This repo's own name and top-level framing (still "toprope-agentdev") are a known-stale leftover of the dropped direction — not yet renamed, deliberately left as-is pending a decision.

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

## Skills

TODO: list skill buckets here as they are created.
- See `skills/SKILLS.md` for the full index.

## References

- `references/diary/` — date/timestamped design diary for the toprope family of products.
- See `references/REFERENCES.md` for the full index.
