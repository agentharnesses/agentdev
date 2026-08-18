---
description: Founding vision for toprope (agentic editor with folder-navigation observability, native agent-harness awareness) and a minimal first plan for getting there.
date: 2026-08-18 09:39 CDT
git:
  toprope-agentdev: 2759c54
  toprope: 21ac2f5
  toprope-cli: 8e842f0
---

## The idea

toprope is meant to be a Claude-CLI-like editor: a desktop app (Electron) that embeds the
Claude Agent SDK to run the agent, wrapped in a VS Code-like interface — a file tree,
editor panes for opening and editing documents, the usual IDE furniture.

The fundamental differentiator is **observability into how the agent navigates folder
structures**. As the agent reads, writes, and moves through files, the user should be able
to *see* that happening against the real folder tree in the UI — not just read it in a
transcript. Navigation and file access become visible, live state, not just log lines.

A specific and important case of this: toprope should support the **agent-harnesses**
standard natively. When a user opens a directory, toprope should detect and list the
harnesses and sub-harnesses present in that directory (HARNESS.md, leaf-typed skill/ref
directories, routing indexes — the same structure this meta-repo's own harness uses), and
let the user visualize and navigate that structure directly, rather than it being something
only the agent understands.

## Where the code actually is right now

Checked the `toprope` and `toprope-cli` repos as of this entry:

- `toprope`: bare three-process Electron scaffold (electron-vite + React 19 + TS). `main`
  owns a `cli-bridge.ts` that spawns a child process and relays stdout/stderr over IPC;
  `preload` exposes a narrow `window.toprope` bridge; `renderer` is an unstyled `App.tsx`
  placeholder. No file tree, no editor, no Claude Agent SDK integration yet. The CLI
  command/protocol is explicitly a placeholder (see `toprope/docs/ARCHITECTURE.md`).
- `toprope-cli`: just a stub README. No implementation yet.

So there's no existing UI or agent loop to build observability *on top of* — the file tree
and editor are themselves unbuilt. That reframes "first order of business": before
observability features mean anything, there needs to be something to observe against.

## Minimal plan

Ordered so each milestone is independently useful and de-risks the next one, rather than
trying to build the full vision in one pass.

1. **File tree + editor shell (no agent).** Renderer gets a real VS Code-style explorer:
   open a folder, render its tree, click a file to open/edit it in a pane. This is pure UI
   plumbing but it's the substrate everything else visualizes against — can't show "the
   agent touched this file" without a tree to highlight it in.

2. **Harness awareness on folder open.** Port the detection logic that already lives in
   this meta-repo's `agent-harnesses` skill (`.harnessleaf` / `.leaf-detectors` / routing
   filename conventions — see `.claude/skills/agent-harnesses/scripts/`) into toprope
   itself, so opening a directory in the app can identify HARNESS.md roots, leaf-typed
   directories, and routing indexes, and annotate the tree with them. This is the "list
   harnesses and sub-harnesses" requirement, done before any agent is involved.

3. **Agent SDK wired in, minimal tool loop.** Embed the Claude Agent SDK in `main`, give it
   a small set of filesystem tools scoped to the open folder, get a basic chat/agent panel
   working end-to-end. Keep the tool surface minimal (read/write/list) rather than trying
   to route everything through `toprope-cli` immediately — see open question below.

4. **Live navigation overlay.** As agent tool calls fire, emit IPC events tagging which
   paths were read/written/listed, and highlight those nodes live in the file tree from
   step 1. This is the core observability feature the project is named for.

5. **Harness-structure visualization.** Extend the step-2 annotations so the user can see
   harness/sub-harness boundaries as a distinct layer in the tree (not just flat files),
   and click into a harness node to see its routing description — surfacing the same
   structure the agent itself uses when it does progressive disclosure.

## Open questions (not resolved here, just flagged)

- **Agent SDK tools vs. toprope-cli.** The current READMEs say the Agent SDK "gives it
  tools to drive" the companion CLI — implying file operations might go *through* the CLI
  rather than direct Node `fs` calls in `main`. Milestone 3 punts on this: start with
  direct fs tools for speed, decide the CLI's actual role once it has more than a stub.
- **Editor scope.** "VS Code-like" could mean anything from a plain textarea to a full
  Monaco integration with syntax highlighting, multi-tab, diffing, etc. Milestone 1 should
  start as simple as possible (plain text editing) and grow only as real needs surface.

## Follow-up: how to actually observe navigation (milestone 4 shape)

Bash is the agent's main point of entry in practice, so observability needs to account for
it rather than only watching typed file tools — but bash shouldn't be the *primary* signal,
since better signal already exists for free.

- Typed SDK tool calls (`Read`/`Write`/`Edit`/`Glob`/`Grep`) give exact paths with no
  parsing — this is the reliable backbone of the file-tree overlay.
- Bash calls need to be intercepted too, but split in two: harness-standard commands
  (`disclose.py` and friends) already print structured JSON, so recognizing those
  invocations and parsing their stdout gives high-fidelity harness-navigation events
  (sessions, selections, resources) — actually the best channel for the harness-standard
  half of the vision. Generic bash (`ls`, `find`, `cat`, `grep -r`, `cd`, ad-hoc scripts)
  only supports heuristic argv parsing, which is inherently fuzzy, so it should render as
  lower-confidence than typed-tool or harness-JSON events.
- Longer term, giving the agent a dedicated SDK tool that wraps harness disclosure (instead
  of it always going through raw `Bash disclose.py ...`) would convert the highest-value
  navigation path from "parse bash output" to "structured tool call" by construction.
- One unified event stream (path, action, confidence, source) should feed the tree overlay;
  harness-session events likely need their own overlay layer rather than being flattened
  into plain path highlights, since they carry session/selection structure a flat list
  can't represent.
