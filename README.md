# agentdev

Development harness for tooling built around the agent-harnesses standard. Holds:

- `agent-harnesses`, `skills/`, `references/` — the shared `agentdev` harness: agent-harness-standard conventions and integration knowledge

The `toprope` Electron app and `toprope-cli` companion CLI submodules that this repo originally held have been removed — that product direction was dropped in favor of a single lightweight VS Code extension, `ahar-visualizer`, developed in its own repo for now (see `references/diary/2026-08-18-1007-vs-code-base-or-extension.md` for why). This repo was renamed from `toprope-agentdev` to `agentdev` to drop that stale product branding (see `references/diary/2026-08-18-2345-toprope-agentdev-renamed-to-agentdev.md`).

See `HARNESS.md` for how Claude should use this harness.
