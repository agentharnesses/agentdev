# agentdev

Development harness for tooling built around the agent-harnesses standard. Holds:

- `agent-harnesses`, `skills/`, `references/` — the shared `agentdev` harness: agent-harness-standard conventions and integration knowledge
- `ahar-visualizer/` (submodule) — a VS Code extension that visualizes an agent-harnesses-standard directory tree and live-tails a `claude` CLI session's navigation, including multiple independently-configured panels side by side
- `traversal-compare/` (submodule) — a comparison-test framework that sandboxes `claude` CLI sessions against paired with/without-harness fixture repos to measure exploration efficiency and consistency, visualized live via `ahar-visualizer`
- `vendor/` — the `agentharnesses`, `cli`, and `metaskill` repos, vendored as submodules

The `toprope` Electron app and `toprope-cli` companion CLI submodules that this repo originally held have been removed — that product direction was dropped in favor of the lightweight VS Code extension above (see `references/diary/2026-08-18-1007-vs-code-base-or-extension.md` for why). This repo was renamed from `toprope-agentdev` to `agentdev` to drop that stale product branding (see `references/diary/2026-08-18-2345-toprope-agentdev-renamed-to-agentdev.md`).

See `HARNESS.md` for how Claude should use this harness.
