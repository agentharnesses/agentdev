---
description: Incident — closing VS Code mid-session kills the Claude Code CLI session running inside it, silently dropping work that hadn't been committed. The ahar-vsvis harness-tree MVP from that session was recovered intact from the ephemeral scratchpad and committed. New harness rule added as a result.
date: 2026-08-18 11:30 CDT
git:
  toprope-agentdev: 704c1ff (dirty — ahar-vsvis submodule pointer bump pending)
  ahar-vsvis: fe8d5b9
---

## What happened

The session that built the `ahar-vsvis` proof-of-concept (harness-aware directory tree
view: `src/harness.ts`, `src/harnessTreeProvider.ts`, `src/extension.ts`, plus
`package.json`/`tsconfig.json`/etc.) was running as a `claude` CLI process inside a VS Code
integrated terminal. Daniel closed the VS Code window to move on, which killed the terminal
and, with it, the Claude Code session mid-work — no warning, no chance to save state, no
diary entry written. From the outside this just looked like "the ahar-vsvis work vanished,"
prompting this entry once the follow-up session went looking for what was lost.

## What was actually lost vs. recovered

Turns out nothing was truly lost — it just hadn't been committed anywhere durable yet. The
interrupted session had built the extension code inside its own **scratchpad directory**
(`/private/tmp/claude-501/.../scratchpad/ahar-vsvis`, a clone of the real `ahar-vsvis` repo),
compiled it successfully, and was in the middle of verifying it actually renders in a real
VS Code Extension Development Host (debugging a workspace-folder-not-opening launch issue)
when the session died. The scratchpad directory itself survived on disk — scratchpads aren't
deleted on session end, only on a fresh `/private/tmp/claude-501/<pid>/...` cycle — so the
next session found it, read the interrupted session's own transcript
(`~/.claude/projects/.../f8d5c6d8-....jsonl`) to reconstruct exactly what had been decided
and built, and copied the code back into the real tracked `ahar-vsvis` checkout.

Recovered and now committed to `ahar-vsvis` (`fe8d5b9`):
- A "Harness Structure" tree view in the Explorer sidebar, rendering the full workspace
  directory tree.
- `HARNESS.md`/routing files/leaf directories visually distinguished with icons, using
  detection logic ported by hand from the `agent-harnesses` skill's `.harnessleaf`/
  `.leaf-detectors` conventions.
- A "Flatten Harnesses" toggle command to collapse down to just harness-relevant nodes.
- Verified in this follow-up session: compiles clean, and — unlike the interrupted session,
  which got stuck fighting a workspace-open launch flag — actually activates and registers
  `aharVsvis.harnessTree` in a real Extension Development Host (confirmed via the host's own
  verbose log line: `MainThreadTreeViews#$registerTreeViewDataProvider aharVsvis.harnessTree`).

## New harness rule: don't close VS Code while a Claude Code session is running in it

Added to `HARNESS.md`. The mechanism: the `claude` CLI is a normal child process of the
VS Code integrated terminal; closing the window (or the terminal pane) sends it a hangup
signal same as closing any terminal tab would. There's no session-end hook that fires in
time to persist state — it's just gone. This is a **process management gotcha**, not
anything specific to this harness's structure, but it bit hard enough (an hour of confusing
"where did the work go" before this recovery) that it's worth a standing warning rather than
tribal knowledge. General mitigation for next time, beyond "just don't do it": prefer
finishing a coding sub-task to a real commit in the tracked repo before ending a VS Code
session, rather than leaving durable-looking work sitting only in the ephemeral scratchpad.
