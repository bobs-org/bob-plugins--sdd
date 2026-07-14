---
create_time: 2026-07-14 06:49:54
status: done
prompt: 202607/prompts/pomodoro_move_only_hash_keymap.md
tier: tale
---

# Plan: Pomodoro Move-Only `#` Keymap

## Goal and behavior

Add a normal-mode `#` mapping to the `task-status-cycler` Obsidian plugin that marks eligible Pomodoro task-link bullets
as move-only entries for the existing Ctrl+Enter completion flow.

- A bare `#` considers only the cursor line.
- An explicit `N#` considers the cursor line plus the next `N` physical lines, matching the requested `1#` example by
  covering two lines total.
- The action is eligible only when the cursor starts on a standalone, non-embedded block-link bullet inside the
  sub-bullet block of an open top-level Pomodoro in `## Pomodoros`.
- Within the bounded line range, append `#` only to similarly eligible links that resolve to real Obsidian Tasks-plugin
  tasks (checkbox lines carrying `#task`). Leave prose, embedded/transcluded links, mixed-content bullets, malformed or
  stale links, non-task block targets, fenced examples, already-marked links, and lines outside the owning Pomodoro
  unchanged.
- Never cross the end of the current Pomodoro's sub-bullet block. Preserve link aliases, indentation, line text, and
  trailing whitespace by inserting the directive immediately after `]]` and before any trailing spaces. Keep the cursor
  at its original logical position.
- On lines where the mapping is not eligible, make no document change. This intentionally assigns normal-mode `#` to
  this workflow instead of Vim's default backward word-search motion.

## Implementation

1. Extend `plugins/task-status-cycler/main.js` with a small, testable planner for the move-only marking range. Reuse the
   existing Pomodoro section/range detection and strict bare block-link parsing so the new producer and the existing
   Ctrl+Enter consumer agree on the exact `[[...#^...]]#` format. Make marking idempotent and return explicit per-line
   replacements rather than mutating while classifying.

2. Add runtime validation for each planned block link through the plugin's existing same-file/cross-file Obsidian link
   resolution path. Require the resolved block to be a proper `#task` checkbox, regardless of its current checkbox
   status, and apply only the validated replacements to the live editor. Restore the original cursor after the batch and
   tolerate one stale or unreadable target without blocking other eligible lines.

3. Register a CodeMirror Vim normal-mode action for `#` in `registerVimMappings`. Distinguish a bare action from an
   explicitly typed count using `repeatIsExplicit`: no count means zero additional lines, while explicit count `N` means
   `N` additional lines. Dispatch the asynchronous editor operation through the active Markdown view/file and contain
   failures so the mapping cannot surface an unhandled rejection.

4. Expand `scripts/test-task-status-cycler.cjs` with focused coverage for:
   - the exact requested two-link `1#` transformation and the bare-`#` single-line case;
   - mapping registration plus explicit-versus-default repeat interpretation;
   - same-note, cross-note, aliased, and trailing-whitespace links that resolve to proper Obsidian tasks;
   - idempotence for existing trailing directives and preservation of cursor/text formatting;
   - rejection of embedded, retired, mixed/prose, multiple-link, malformed, fenced, outside-section, completed-Pomodoro,
     stale-target, and non-task-target cases;
   - range clamping at the current sub-bullet block and best-effort behavior when only some counted lines resolve.

5. Release and document the feature in the `bob-plugins` source of truth: bump the `task-status-cycler` manifest minor
   version, update its manifest/README description and plugin table entry to mention the `#` marking keymap and counted
   range semantics, and keep the existing Ctrl+Enter move-only description aligned.

## Verification and deployment

1. Run the focused task-status-cycler test file, then the complete `npm test` suite and `npm run validate`
   manifest/syntax checks from the `bob-plugins` checkout.
2. Preview deployment with `bob plugins sync --no-pull --dry-run -p task-status-cycler -r .`, then deploy the validated
   plugin with `bob plugins sync --no-pull -p task-status-cycler -r .` so the vault receives the source-of-truth
   `main.js` and manifest without touching plugin settings.
3. Confirm the deployed keymap manually in Obsidian with the supplied Pomodoro example: `1#` marks both task-link
   bullets, leaves the prose bullet unchanged, and subsequent Ctrl+Enter moves clean links forward without adding tomato
   history for those entries.
