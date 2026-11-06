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

### 2. Mac-only build workflow
Status: derived from upstream `cd.yml`, **not yet run**.
File: `.github/workflows/mac-build.yml` (new file, so it never conflicts).
Manual only (Actions tab > Mac build (fork) > Run workflow). Builds just
`Oak-macOS-arm64.dmg` and uploads it as the `oak-macos` artifact.
Checkout uses `submodules: recursive` (upstream's own checkout steps do not fetch
the quercus-engine submodule, so their CI fails on forks).
If upstream changes the macOS steps in `cd.yml`, copy those steps across.

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

### 3. Cargo.lock refresh
Upstream's `Cargo.lock` at `8b7b9af` is one line behind (`windows-sys 0.61.2`
missing from the quercus-engine-rs entry), so `cargo build --locked` fails.
The fix is that one added line. When upstream commits a fresh lockfile, take
theirs on conflict (`git checkout --theirs Cargo.lock`) and drop this entry.

## Ideas / not done yet
- Native macOS menu bar (upstream only draws an in-window bar).
- Show import errors in the UI.
- Find out why playback is slow on M1 (hardware decode, proxies, ACEScg).

## 5. macOS app launch crash: engine dylib not bundled
- File: `.github/workflows/mac-build.yml` (Package (macOS) step)
- Copies `libquercus_engine.dylib` into `Contents/MacOS` before `bundle-dylibs-macos.sh` (which re-signs everything).
- Merge note: if upstream's cd.yml starts bundling the engine itself, drop this.

## 6. Keyboard presets (Premiere-style, Resolve-style)
- Files: `presets/shortcuts/*.shortcuts` (no code changes). Import via Preferences > Keyboard > Import.
- Oak's defaults are already mostly Premiere-like; the presets only list the differences.

## 7. Ruler drag scrubs the playhead
- File: `crates/oak-app/src/widgets/timeline/timeline_view.rs` (`update_ruler_drag`)
- Plain drag on the ruler now seeks continuously; Shift+drag keeps upstream's work-area reshape.
- Merge note: re-apply if upstream touches `update_ruler_drag`.

## 8. Playback timing logs
- Files: `crates/oak-app/src/oakui/renderops.rs` (`to_display`), `crates/oak-app/src/oakui/real.rs` (`preview_slot_frame`)
- Logs per-stage frame display time as `[fork] frame ... display ms: ...` and `[fork] gpu register ms` at info level (run with `RUST_LOG=info`).
- Diagnostic only; remove once the slow stage is fixed.

## 9. Surface swallowed render failures
- File: `crates/oak-app/src/oakui/real.rs` (preview drain, preview submit, `full_res_worker`)
- Failed/unstartable renders were silently dropped (viewer then shows the color-bar placeholder); now logged at warn as `[fork] ... render failed`.

## 10. Preview-window stats log + faster colour transform
- `crates/oak-app/src/oakui/real.rs`: once-a-second `[fork] preview window: playhead N ready N rendering N` (info).
- Engine fork commit `colormath: split mid-size frames across threads`: `par_bands` in `quercus-core/src/colormath.rs` parallelises from ~16k px (was 2^19 px, which left 960x540 single-threaded at ~47 ms).
- Finding (not yet fixed): the timeline preview path (`apply_montage_effect` in `quercus-render/src/eval.rs`) only evaluates built-in Opacity and OFX plugins; all other built-in effects (Invert, Three-way colour, VHS REVRB) pass through unchanged.

## 11. Effects/text in the preview: graph-renderer fallback
- Engine fork: `quercus-engine/src/preview_video.rs` + `preview_video/montage.rs` (`sequence_needs_graph_render`).
- If a visible clip has a built-in effect other than Opacity, or is not footage (text, solid), the preview renders the frame with the full node-graph renderer instead of the montage fast path.
- Set `QUERCUS_NO_GRAPH_FALLBACK=1` in the environment to turn the fallback off (compare speed).
- Merge note: if upstream makes the montage path evaluate all built-in effects, drop this fallback.
- Fallback also triggers for visible transitions, clip speed != 1 and reversed clips (not represented in the montage).
