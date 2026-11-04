# Fork changes

This file lists every change this fork makes on top of upstream
(`OakVideoEditorCommunity/oak`). Keep it up to date: one entry per change,
newest first. It is the checklist to walk through after pulling upstream.

Upstream base: `8b7b9af` (Oak 0.5.2-alpha era, `main`)
Branch with our work: `mac-fixes`
Debug logs from our changes start with `[fork]`. See them with:
`RUST_LOG=info /Applications/Oak.app/Contents/MacOS/oakapp 2>&1 | grep '\[fork\]'`

## How to keep in sync with upstream

    git fetch upstream
    git checkout main && git merge --ff-only upstream/main && git push origin main
    git checkout mac-fixes && git merge main     # or: git rebase main
    # resolve conflicts using the entries below, then build and test

## Changes

### 1. Auto-create project + sequence on file drop / import
Status: written, **not yet built or tested** (needs the project FFmpeg build).
Why: dragging a file onto the window or using Import did nothing unless a
project was already open (error only went to the console).
Behaviour:
- A file dropped anywhere in the window (not on the project panel, which
  keeps its import-only handler) is imported and placed on the timeline at
  the playhead. Several files are placed back to back.
- Import (button, menu, drop) creates a project when none is open.
- The startup project manager closes on drop. The sequence is created by the
  existing `drop_footage` path.
Files touched (look here first if an upstream merge conflicts):
- `crates/oak-app/src/oakui/engine.rs` - new trait method `import_footage_entry`
- `crates/oak-app/src/oakui/real.rs` - `import_footage` now calls
  `import_footage_entry`, which creates a project if needed
- `crates/oak-app/src/app.rs` - `OakApp::import_and_place` and an
  `.on_drop(ExternalPaths)` handler on the root div in `Render`
Known gaps: import errors are only logged; clip always lands on video track 1.
If upstream adds its own drop-to-create behaviour, drop this change.

## Ideas / not done yet
- Native macOS menu bar (upstream only draws an in-window bar).
- Show import errors in the UI.
- Find out why playback is slow on M1 (hardware decode, proxies, ACEScg).
