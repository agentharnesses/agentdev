---
description: How to safely launch a throwaway Extension Development Host to eyeball changes to the ahar-visualizer extension — or to get a working live-visualization target for traversal-compare, since the vscode:// URL fallback silently fails against a stale main-window install — without risking the Claude Code session that's usually running inside the "real" one.
---

## The hazard

This Claude Code session very often runs *inside the integrated terminal of
the same Extension Development Host window* used to test ahar-visualizer —
i.e. `code --extensionDevelopmentPath=.../ahar-visualizer <agentdev>`
is the window hosting the shell this session's tool calls run in. Reloading,
closing, or quitting that window kills the session mid-work, the same way it
did in the incident documented in `diary/2026-08-18-1130-vscode-close-killed-session-mvp-recovered.md`.
Always check the process ancestry (`ps -o pid,ppid,command -p $$` and walk
`ppid` up) before touching any running VS Code window — if a `Code Helper`
or `Code --extensionDevelopmentPath=...ahar-visualizer` process shows up in
the chain, that's the window this session lives in; don't reload/close it.

## Automation is a dead end here — don't try

macOS's Accessibility layer (`osascript`/System Events) only ever exposes
**one** `"Code"` GUI process to script against, no matter how many separate
Electron instances of VS Code are actually running at the OS level (verified
2026-08-18: three simultaneous `code --user-data-dir=...` instances running,
`tell application "System Events" to every process whose name is "Code"`
only ever returned the first-launched one's PID). That first-launched one is
reliably the real session-hosting window. So there is no safe way to send
keystrokes or window-activate a second instance specifically — any attempt
risks typing into the real window instead. Don't use `osascript`
keystroke/activate calls against a new dev-host instance for this reason.

## The safe pattern

1. Launch a genuinely separate, disposable instance with its own profile:
   ```
   code --extensionDevelopmentPath=/path/to/ahar-visualizer \
        --user-data-dir=/tmp/ahar-visualizer-demo-N \
        --skip-welcome --skip-release-notes \
        /path/to/agentdev
   ```
   Use a fresh `/tmp/ahar-visualizer-demo-N` each time (increment `N`) to
   avoid colliding with a still-running earlier instance.
2. **Don't pass `--new-window` alongside the folder path.** That combination
   silently failed to load the workspace (opened to an empty welcome screen
   with no folder, no explorer content) when tested here. Omitting
   `--new-window` and relying on the fresh `--user-data-dir` to force a new
   window worked correctly and loaded the folder as expected.
3. To check it loaded: poll `ps aux | grep -- "--user-data-dir=/tmp/ahar-visualizer-demo-N"`
   for the main Electron process, not System Events — `ps` sees every
   process regardless of what Accessibility will expose.
4. To look at it: a plain `screencapture -x <path>` of the whole screen is
   enough, and needs no window targeting at all. `HarnessTreePanel.createOrShow()`
   already runs unconditionally in `activate()`, and Activity Bar icons
   appear without any click — so the tree panel and any new sidebar icon are
   already visible on screen the moment the window finishes loading, no
   command-palette or button click required first.
5. Beyond that first screenshot, hand it back to Daniel to click around
   himself rather than trying to drive it further — he's right there, and
   it sidesteps the automation dead-end above entirely.

## Also the standard way to get live `traversal-compare` visualization

This isn't only for eyeballing changes to `ahar-visualizer` itself — it's also the documented,
working way to watch a `traversal-compare run` (or `traversal-compare view <run-id>`) live, per
`traversal-compare/skills/running-a-test/SKILL.md` step 0. Reason: `viewer.py` prefers the
dev-queue channel (below) and only falls back to a `vscode://` URL when no dev-host heartbeat is
live, and that URL fallback silently fails whenever the main window's installed `ahar-visualizer`
build predates whatever handler is being fired at it (see the section below) — which is the normal
state, since there's no auto-update (`ahar-visualizer/references/release-process.md`). So: launch
the dev-host with the safe pattern above (workspace path doesn't matter beyond "some real
directory," `agentdev` itself is fine), confirm `/tmp/ahar-visualizer-dev-queue/.heartbeat` is
fresh (<3s old), *then* run or view the comparison — no reinstalling or reloading the real window
needed. Verified end-to-end 2026-08-19 (`diary/2026-08-19-1230-...` for the dev-queue build-out;
confirmed again live the same day replaying a completed `efficient-exploration` run into a fresh
dev-host).

Two `run` flags matter specifically for getting this right, both confirmed live 2026-08-19: leave
`--variant` at its default (`both`) — a single-variant run only ever opens one panel, not the
side-by-side pair that's the actual point of watching live — and never add `--ephemeral` to a run
you intend to watch or inspect afterward, since it deletes each `sandbox-<variant>/` right after
grading, out from under a panel that may still be open and pinned to that exact path (the panel
goes stale/empty the moment the run finishes, not just "nothing left to replay via `view` later").
See `traversal-compare/skills/running-a-test/SKILL.md` step 2 for the fuller writeup.

Once a dev-host is up and a comparison is running against it, this session's own automation (see
"Automation is a dead end here" above) can confirm each panel's request landed — `run`/`view` print
`[viewer] <label>: dev queue` per variant — but cannot reliably screenshot the dev-host window
itself: macOS Accessibility exposes only the first-launched `Code` GUI process to scripting even
when a second (the dev-host) is genuinely running (confirmed live via `osascript ... every process
whose name is "Code"` returning both processes to `ps` but no reliable way to screenshot the
second specifically), so a `screencapture -x` of the whole screen just captures whichever window
already has focus — normally the real session-hosting window, not the dev-host. Hand the visual
check back to Daniel for this reason, same as step 5 of the safe pattern above.

## `vscode://` URLs have the same limitation — confirmed, and worked around

Firing a `vscode://` URL (e.g. `open "vscode://agentharnesses.ahar-visualizer/openTree?..."`,
`ahar-visualizer`'s scriptable multi-panel hook — see its own
`references/multi-panel-testing.md`) is a *different* mechanism from the Accessibility/`osascript`
one above, but was confirmed (2026-08-19) to have the identical failure mode: macOS routes
`vscode://` to the single registered `Code` app bundle, not to whichever `--user-data-dir`
instance is "meant" to receive it. Tested directly: with a disposable dev-host instance running
per the pattern above, firing the URL still opened a permission dialog in the *real*
session-hosting window, never the dev-host — and even after allowing it, nothing happened there
either, since its installed extension predated the handler being tested. So a `vscode://` URL
cannot be used to drive a dev-host from a script; don't attempt it.

The fix, if a dev-host specifically needs to be scriptable (not just eyeballed): `ahar-visualizer`
now has a dev-mode-only file-queue alternative (`src/devQueue.ts`) that sidesteps the OS/URL layer
entirely — a script writes a small JSON request file to a fixed directory instead of firing a URL,
and the dev-host (only when `context.extensionMode === vscode.ExtensionMode.Development`) polls
for it directly. Verified working end to end: a `traversal-compare` comparison run correctly
landed its two live panels in a disposable dev-host, never touching the main window. Full details
in `ahar-visualizer/references/multi-panel-testing.md`'s "Dev-mode: reaching a dev-host directly"
section — nothing further needed here beyond knowing it exists and why.

## Cleanup

Each instance leaves behind a real `/tmp/ahar-visualizer-demo-N` profile
directory and a handful of Electron helper processes. They're disposable —
`rm -rf` the directory once done, or just let old ones accumulate under
`/tmp` (they don't self-clean, but they're harmless clutter, not a leak that
affects anything else).
