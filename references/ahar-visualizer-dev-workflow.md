---
description: How to safely launch a throwaway Extension Development Host to eyeball changes to the ahar-visualizer extension, without risking the Claude Code session that's usually running inside the "real" one.
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

## Cleanup

Each instance leaves behind a real `/tmp/ahar-visualizer-demo-N` profile
directory and a handful of Electron helper processes. They're disposable —
`rm -rf` the directory once done, or just let old ones accumulate under
`/tmp` (they don't self-clean, but they're harmless clutter, not a leak that
affects anything else).
