---
description: Resolved — toprope branding dropped entirely; the project becomes ahar-vsvis, a single VS Code extension visualizing the agent-harnesses standard (not a bespoke editor, VS Code fork, or companion CLI). Includes a survey of open-source AI code editors, a verified spike proving the hook-registration + transcript-tailing observability design against a real Claude Code CLI, and an honest risk assessment of which parts of that design are solid vs. provisional.
date: 2026-08-18 10:07 CDT
git:
  toprope-agentdev: 2759c54
  toprope: 21ac2f5 (dirty — milestone 1 file-tree/editor scaffold, uncommitted)
---

## The question

After getting milestone 1 (file tree + editor shell) rendering, Daniel's instinct: a lot of
what toprope needs — file explorer, editor panes, git integration, window management — is
functionality VS Code already has. Building it bespoke in Electron risks reinventing a lot
of wheel for UI plumbing that isn't the actual differentiator. Should toprope be built on
top of VS Code instead of from scratch?

## My take: extension, not a fork

Three ways to "use VS Code as a base," in increasing order of commitment:

1. **VS Code extension** — ship toprope as an extension (or extension pack) that runs
   inside stock VS Code. Get the editor, explorer, git integration, and window management
   for free. Build the actual differentiator — agent-navigation observability, harness
   detection — via extension APIs: `FileDecorationProvider` for live highlight badges on
   files the agent touches, a custom `TreeDataProvider` for harness/sub-harness
   visualization, `WebviewView` for anything needing a fully custom panel (a harness-session
   graph, say). Bound by the extension API's surface — can't rewrite core window chrome or
   editor internals, can't get as deep as weaving agent behavior into core editing itself.
2. **VS Code fork** — clone Code - OSS and modify it directly. Full control over core UX,
   but commits to permanently syncing upstream and maintaining a large codebase indefinitely.
   This is what Cursor and Windsurf did — they wanted AI woven into core editor behavior,
   not just an add-on panel, which extension APIs don't allow.
3. **Bespoke Electron app** (the current path) — full control, but reinventing editor, file
   tree, and git UI from scratch, which milestone 1 is already surfacing as real surface
   area (see the preload-path bug hit while testing it — `references/diary/2026-08-18-0939-vision-and-minimal-plan.md`).

Recommendation: start as an **extension**. It reuses everything VS Code does well, focuses
effort entirely on the observability layer (the part nobody else has built), and is a small
enough bet to reverse if the extension API turns out to be too limiting. A fork is the right
move only if the ambition is genuinely core-editor-level AI integration, not an add-on panel
— worth revisiting once/if extension APIs are actually hit as a ceiling, not before.

Direct consequence if this is taken: the milestone 1 scaffold just built (FileTree, Editor,
FsBridge, the IPC plumbing) gets thrown away — VS Code already has all of that; an extension
would only add decorations/views on top of the existing explorer/editor, not reimplement them.

## Survey: existing open-source AI code editors

Asked to check what's already out there, since it bears directly on fork-vs-extension-vs-
bespoke. Three real architectural camps exist:

- **VS Code forks.** [Void](https://github.com/voideditor/void) is the clearest example —
  MIT-licensed VS Code fork (Void's own additions Apache 2.0) adding Cursor-style inline
  edits, agent mode, and autocomplete without a proprietary backend. Notably: **Void is now
  deprecated and no longer accepting contributions** — a real data point on how hard the
  fork-maintenance burden is to sustain, even for a well-funded, well-starred project.
  Cursor and Windsurf took the same fork route but stayed closed-source and commercially
  funded, which is likely why they've sustained it where Void didn't.
- **VS Code extensions.** This is the thriving camp. [Cline](https://github.com/cline/cline)
  — a fully open-source, agentic VS Code extension — crossed 58k GitHub stars and 5M+
  installs by early 2026, per GitHub's 2025 Octoverse report (fastest-growing open-source
  project). It does full agentic multi-file editing, terminal command execution with
  approval, and works with any model. [Roo Code](https://github.com/RooCodeInc/Roo-Code)
  forked Cline to add multi-agent modes and better large-codebase handling (~24k stars,
  though the project shipped its final release and archived in May 2026). Kilo Code then
  forked both, combining Cline's stability with Roo's multi-mode system plus inline
  autocomplete and an orchestrator mode, backed by an $8M seed round. This whole lineage is
  strong evidence the extension route scales to real agentic capability without needing to
  own the editor shell.
- **Built from scratch, non-VS Code.** [Zed](https://zed.dev) is the standout: written
  entirely in Rust with its own GPU-rendered UI framework (GPUI) rather than a browser/
  Electron foundation, structured as 235+ crates. GPL (editor) / AGPL (server) / Apache 2.0
  (GPUI). Notably, Zed authored the **Agent Client Protocol (ACP)** — an open standard for
  connecting AI agents to editors, already used to plug in Claude Agent, Codex CLI, and
  others. Worth a closer look later: if toprope ever needs a standard wire format between
  "the agent" and "the editor," ACP may already be the right shape to adopt or interop with,
  rather than inventing toprope's own protocol from scratch.

## Where this leaves things — resolved: CLI + VS Code extension

Daniel reframed the actual mission, which settles the question above: toprope isn't meant
to be a daily-driver editor, at least not now. Its fundamental purpose is to **demonstrate
the value of the agent-harnesses standard** — rich enough visualization and interaction with
harnesses that other tools/editors have a reason to consider adopting the standard
themselves. That's a demo/reference-implementation mission, not a "build a product people
use every day" mission, and it argues for a deliberately **limited scope of responsibility**
rather than maximal feature surface.

Given that framing, the decision: a **robust CLI tool** (`toprope-cli`) paired with a
**VS Code extension** that works alongside it is sufficient — no separate bespoke editor
needed at all. This resolves the fork-vs-extension question above in favor of extension
(confirmed, not just recommended), and goes further: it also rules out continuing the
bespoke Electron shell as the primary product, since an extension already covers the "rich
visualization inside an editor" half of the mission without reimplementing an editor.

Open implication worth surfacing, not yet resolved: this changes what the `toprope`
Electron app (the milestone 1 scaffold — FileTree, Editor, FsBridge) is *for*. Either it
gets retired in favor of `toprope-cli` + a new VS Code extension, or it's kept around for
some other reason (e.g. a standalone demo shell independent of VS Code). Worth asking
directly rather than assuming.

## Further narrowed — drop the companion CLI too, extension only

Follow-up: why build `toprope-cli` at all? Claude Code (the `claude` CLI) already works
well as the agent runner — a bespoke companion CLI would just be duplicating that role.
The extension doesn't need to wrap or reimplement an agent CLI; it needs to observe and
visualize what's happening as Claude Code operates in a VS Code workspace.

So the scope shrinks again: the actual deliverable is **a single VS Code extension**, built
on top of the already-existing, already-working Claude Code CLI, demonstrating
agent-harnesses-standard visualization and interaction. Neither the bespoke `toprope`
Electron app nor a bespoke `toprope-cli` companion CLI are needed for the core mission.

Two open implications this leaves:
- What happens to the `toprope` and `toprope-cli` submodules/repos in this meta-repo —
  retired, repurposed, or left alone for now while the extension gets built elsewhere?
- How does the extension actually observe Claude Code's activity if it isn't the one
  spawning/driving the agent process? Investigated below.

## Investigated: how would the extension actually observe Claude Code?

Checked before committing to dropping `toprope-cli` — verified directly against this very
session rather than trusting docs alone, since docs can drift. Two real, already-working
surfaces:

1. **Session transcripts.** Every Claude Code session writes a live JSONL file to
   `~/.claude/projects/<project>/<session-id>.jsonl`. Pulled real records from this
   session's own transcript file (2.5MB+ and growing as this conversation ran): `Read`/
   `Edit`/`Write` tool_use blocks carry a structured `file_path` field directly — no bash
   parsing needed, confirming the Layer 1 idea from the earlier navigation-observability
   follow-up. `Bash` tool_use blocks carry the full command string, and the paired
   tool_result carries the full stdout — so a `disclose.py` invocation's structured JSON
   output (session id, selected items, resources) is sitting right there too, confirming the
   Layer 2 harness-specific-parsing idea. An extension just needs to tail this file: zero
   configuration, works retroactively, no permission changes required from the user.

2. **Hooks.** `PreToolUse`/`PostToolUse` fire a script on every tool call with structured
   JSON on stdin: `session_id`, `prompt_id`, `transcript_path`, `cwd`, `tool_name`,
   `tool_input`, and (PostToolUse only) `tool_response`. Lower latency than tailing, and
   `PreToolUse` fires *before* the tool executes, which transcript-tailing alone can't give.
   Costs more setup: the extension has to write hook entries into `.claude/settings.json` on
   activation, which is a more invasive first-run step than just watching a file.

One more data point, pointing the opposite direction: the official Claude Code VS Code
extension runs a local WebSocket server, discoverable via a lock file — found this session's
own live one at `~/.claude/ide/54020.lock`, containing
`{pid, workspaceFolders, ideName, transport: "ws", authToken}`. That's how the `claude` CLI
running in a VS Code integrated terminal connects back to the IDE to call tools like
`mcp__ide__getDiagnostics`. Real precedent for local-loopback IDE↔CLI integration — there's
even a community-reverse-engineered reimplementation of it for Visual Studio
([firish/claude_code_vs](https://github.com/firish/claude_code_vs)) — but the direction is
backwards from what toprope needs: it's "IDE exposes tools *to* the agent," not "agent
streams events *to* the IDE." Not directly useful for observability, but confirms this class
of integration is a supported, well-trodden pattern rather than something being invented
from nothing.

**Conclusion: transcript-tailing is the right MVP foundation** (no setup, already proven
against a live session) **with hooks as a later layer** for lower-latency/pre-execution
signals. This resolves the open question above — dropping the custom CLI holds up; Claude
Code already exposes what a purely-observing extension needs.

Still open: what happens to the `toprope` and `toprope-cli` submodules/repos in this
meta-repo — retired, repurposed, or left alone for now while the extension gets built
elsewhere?

## Update: `toprope`/`toprope-cli` retired; hook-registration spike run and verified

Daniel's call: drop `toprope-cli` too — Claude Code itself already works well as the agent
runner, so a bespoke companion CLI would just duplicate it. The whole product becomes a
single VS Code extension (`toprope-vscode`, to be created as a new repo and folded into
this meta-repo later). `toprope` and `toprope-cli` are being removed from this meta-repo.

Before committing further, ran the actual spike on the riskiest piece — registering hooks
from a VS Code extension and having it observe them — empirically, against a real running
Claude Code CLI (v2.1.234), not just docs. Method: a local Node HTTP server logging
incoming POSTs, a `.claude/settings.local.json` in this repo wired to point at it, and
repeated `claude -p --output-format json` subprocess invocations to trigger real
`SessionStart`/`SessionEnd` events.

**Findings:**
- `type: "http"` hooks work — confirmed for `SessionEnd`, which POSTed the exact documented
  payload (`session_id`, `transcript_path`, `cwd`, `hook_event_name`, `reason`) straight to
  the local server, no relay script needed.
- `type: "http"` hooks **do not fire for `SessionStart`** in this CLI version — silently
  dropped, with or without a `matcher` field (tested both). Likely a timing/initialization
  gap specific to how early `SessionStart` fires in process startup, not a general http-hook
  bug (SessionEnd on the same server worked fine).
- `type: "command"` hooks **do** fire reliably for `SessionStart` — confirmed directly, then
  confirmed again as a full working relay: a one-line Node `command` hook that reads the
  hook's stdin JSON and forwards it via `http.request` to the extension's local server. Full
  round trip verified end-to-end for both events in one run.
- **Resume-retroactivity confirmed**, which was the second open question: started a session
  with *no* hooks configured, then added the hook config afterward, then resumed that same
  session (`claude -p -r <session_id>`) — `SessionStart` fired correctly with `source:
  "resume"`, the original `session_id`, and the correct `transcript_path`. So a hook added
  after a session already exists still gets picked up on its next resume/clear/compact/fork,
  not just on brand-new sessions. Caveat: this only covers resume-family events — a session
  that just keeps running continuously without ever resuming wouldn't retroactively fire.

**Design conclusion, revised from the original plan:** use `command`-type (not `http`-type)
for the `SessionStart` hook — a tiny bundled Node relay script, proven reliable — and
`http`-type is fine for `SessionEnd`. Both only fire once per session lifecycle event, so
neither sits in Claude's per-tool-call execution path; the actual tool-call stream still
comes from passively tailing `transcript_path`, exactly as designed before the spike. The
one thing the spike changed is dropping the "no relay script at all" simplification —
`SessionStart` needs one, `SessionEnd` doesn't.

Caveats for whoever builds this next: verified against CLI v2.1.234 specifically — hook
behavior isn't a stable public contract across versions, so re-verify if this breaks after a
`claude` upgrade. Also: the four `claude -p` spike invocations cost a small amount of real
API spend (roughly $0.13 total), worth knowing since every spike like this has a real,
if small, dollar cost.

## Is this a hack? Honest risk layering

Daniel pushed back on the spike result before committing further: is the hook-based design
a hack, and is it expected to hold up? Worth separating into three layers with different
risk profiles rather than answering yes/no:

- **Hooks themselves — not a hack, solid ground.** `SessionStart`/`SessionEnd`/etc. are a
  first-class, documented Claude Code feature built explicitly for external tooling to react
  to session/tool lifecycle events. This is the sanctioned use case, not something creative.
- **`command`-type instead of `http`-type for `SessionStart` — a workaround, low risk.** The
  docs don't document any exception for `SessionStart`+`http`; what the spike found looks
  like a bug or an early-startup timing gap. But `command` hooks are the original,
  foundational hook mechanism (`http` is the newer convenience type layered on top), so this
  isn't hacking around hooks — it's picking the more battle-tested of two supported options.
  If Anthropic fixes the gap later, the relay script just becomes deletable, nothing breaks.
- **Transcript-tailing — the actually hacky part, genuinely provisional.** The hook payload's
  `transcript_path` *field* is documented, so the file's existence and location are
  implicitly sanctioned. But the internal JSONL schema per line — the `tool_use`/
  `tool_result` block shapes actually being parsed — isn't published as a versioned public
  contract; it's Claude Code's internal conversation-persistence format (also used for
  `--resume`), with no guarantee it stays stable across CLI versions. This is the piece most
  likely to quietly break on some future `claude` upgrade.

Mitigation, to build in from the start rather than retrofit: keep transcript-tailing as the
MVP data source (proven, cheap, gives structured file paths for free), but isolate it behind
a single narrow module (`onToolCall(event)` or similar) rather than entangling it with the
rest of the extension. If the transcript format ever breaks, there's a known, fully-
documented fallback that only requires rewriting that one module: switch to `PreToolUse`/
`PostToolUse` hooks instead (same lifecycle mechanism, just wired to more events) — slower to
build and sits in the tool-call path, which is why it's the fallback and not the starting
point, but it's the stability-guaranteed version of the same idea.

## Rebrand: toprope → ahar-vsvis

Dropping the `toprope` branding entirely, not just the product direction. New name:
**`ahar-vsvis`** — a new GitHub repo, described as "a lightweight but powerful visualization
tool for ahar-powered repos" (`ahar` = agent-harnesses, the standard this whole project
exists to demonstrate). Being created as its own repo for now, not yet added to this
meta-repo as a submodule (same "maybe later" status `toprope-vscode` had before the rename).

Consequence acted on: the `toprope` and `toprope-cli` submodules are removed from this
meta-repo (`git submodule deinit` + `git rm`, `.gitmodules` cleared). `HARNESS.md` and
`README.md` updated to stop describing them and to point at this entry for why.

Left deliberately undone: this meta-repo's own name (`toprope-agentdev`) and directory are
now a stale leftover of the dropped branding, not renamed yet. Renaming a repo/directory is
more disruptive than editing routing files, so it's flagged here rather than done
unilaterally — worth deciding explicitly once `ahar-vsvis` exists and the shape of "does the
shared harness knowledge move into that repo, or does this meta-repo get renamed to match"
is clearer.
