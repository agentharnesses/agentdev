---
description: Confirmed vscode:// URLs can't reach a disposable Extension Development Host either (same limitation as osascript), and built a file-queue alternative for ahar-visualizer's dev mode — verified working end to end with a real traversal-compare comparison run landing live panels in a dev-host.
date: 2026-08-19 12:30 CDT
git:
  agentdev: b9bf127 (dirty — this entry's own changes not yet committed)
  ahar-visualizer: 7acfe35 (dirty — devQueue.ts changes not yet committed)
  traversal-compare: 53e0c4a
---

## What happened

Tried to actually watch `traversal-compare`'s side-by-side comparison live, as originally
intended. `traversal-compare run`'s viewer step fires `open "vscode://agentharnesses.ahar-visualizer/openTree?..."`
for each variant — but this session runs inside the real, main VS Code window (confirmed via
process ancestry, `ps -o pid,ppid,command -p $$` walking up to a `Code --user-data-dir=.../Application Support/Code`
process), so that URL landing there is exactly the hazard `2026-08-18-1130-vscode-close-killed-session-mvp-recovered.md`
already warned about.

Launched a disposable dev-host per `dev-preview`/`ahar-visualizer-dev-workflow.md`'s safe pattern
and fired a test URI to see where it actually went — screenshotted both before and after. Result:
it went to the main window every time, never the dev-host, and even after allowing the permission
dialog there, nothing happened (the main window's installed extension predates the handler). This
confirms `vscode://` URL routing has the *identical* limitation the dev-workflow doc already
documented for `osascript`/Accessibility: macOS resolves to the one registered `Code` app bundle,
not to whichever `--user-data-dir` instance is "meant" to receive something. Recorded in
`references/ahar-visualizer-dev-workflow.md`.

## The fix

Brainstormed four options (decouple the session from VS Code's terminal; use VS Code Insiders as a
separate app identity; a local file-queue IPC channel; accept manual reloads) and picked the
file-queue — it's the only one with no residual unverified assumption, needs no new software, and
doesn't touch the main window at all.

`ahar-visualizer/src/devQueue.ts`: `DevQueueWatcher` polls a fixed directory
(`/tmp/ahar-visualizer-dev-queue/`) for JSON request files, same shape as the URI handler
(`{rootPath, sessionFile?, label?}`), and touches a `.heartbeat` file on every tick — live or not —
so a caller can distinguish a running watcher from a stale directory a killed dev-host left
behind. Only starts when `context.extensionMode === vscode.ExtensionMode.Development`, so it's
inert for a real install. Callback-based (mirrors `TranscriptWatcher`'s design, not
`HarnessTreePanel`'s), so it's unit-testable without real `vscode` — 8 new tests.

`traversal-compare/src/traversal_compare/viewer.py`: checks the heartbeat's freshness (<3s) before
choosing the queue over the URI; falls back automatically the moment a dev-host goes away. 6 new
tests.

Verified for real, not just unit-tested: launched a fresh dev-host, ran
`traversal-compare run efficient-exploration/add-currency --variant both`, and both panels
appeared in the `[Extension Development Host] traversal-compare` window — visually showing exactly
the phenomenon the framework measures (with-harness: one narrow path; without-harness: three
branches lit up, matching the earlier precision numbers). Screenshot confirmed; main window
untouched throughout.

## Where this leaves things

Both `ahar-visualizer`'s `devQueue.ts` and `traversal-compare`'s `viewer.py` changes are
implemented, tested, and verified live — not yet committed as of this entry. The `ahar-visualizer`
commit from `2026-08-19-1145-...` is still unpushed and the main window's installed extension
still predates all of today's work; none of that changed today, but it no longer blocks live
side-by-side viewing — the dev-host path works around it entirely, for dev purposes.
