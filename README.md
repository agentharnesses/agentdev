# toprope-agentdev

Meta-repo (repo of repos) for the toprope family of products. Holds:

- `toprope/` — the toprope app itself (Electron + TypeScript agentic client), as a git submodule
- `agent-harnesses`, `skills/`, `references/` — the shared `agentdev` harness: development knowledge (SDK integration patterns, Electron conventions, CLI-integration contracts) usable across toprope and its sibling/companion repos

See `HARNESS.md` for how Claude should use this harness.
