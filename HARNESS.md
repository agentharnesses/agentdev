---
name: toprope-agentdev
description: Development harness for the toprope family of repos — an Electron-based agentic client (toprope) and its sibling/companion repos (e.g. the CLI). Load this when working on toprope's source or any sibling repo's source, not when toprope is being used to explore some other target harness.
---

## Upon loading the Harness

toprope is a product: an agentic client, built with Electron + TypeScript, that embeds the Claude Agent SDK and gives it tools to interact with a companion CLI (developed in a separate repo). This meta-repo is a repo of repos: it holds the toprope app and its sibling/companion repos as git submodules, plus this one shared harness — SDK integration patterns, Electron architecture conventions, the agent-harness standard itself (since toprope's UX is expected to follow it), and CLI-integration contracts — usable across all of them.

Product source lives in the sibling repo directories (e.g. `toprope/`), not in this harness — treat this harness as developer knowledge, not application code.

## How to Find Information for Claude

Use the `agent-harnesses` skill to explore the harness just in time, based on prompts from the user. Select only what is relevant and repeat until the session is complete, then read the returned resources.

When **maintaining the harness** (adding, moving, or renaming files), consult the `agent-harnesses` skill for reverse progressive disclosure to keep routing files in sync.

## Skills

TODO: list skill buckets here as they are created.
- See `skills/SKILLS.md` for the full index.

## References

TODO: list reference documents here as they are added.
- See `references/REFERENCES.md` for the full index.
