---
create_time: 2026-07-12 12:19:14
status: done
prompt: 202607/prompts/clear_pomodoro_markers_on_reopen.md
tier: tale
---

# Clear Pomodoro block-link markers on reopen

## Context

Task Status Cycler marks completed Pomodoro block-link occurrences with `🍅`. Reopening a completed Pomodoro currently
reopens its eligible direct task roots, restores retired links, and changes the Pomodoro checkbox from Done to Todo, but
it leaves those completion markers on the Pomodoro's sub-bullets. The existing marker helpers can already remove marker
prefixes without changing link aliases, embedding, strikethrough, whitespace outside the prefix, or line endings; the
missing piece is wiring that normalization into the Pomodoro reopen lifecycle.

Marker clearing should be specific to explicitly reopening a Pomodoro. The vault-wide reference-restoration transform
should continue to preserve Pomodoro history markers when an ordinary referenced task is reopened elsewhere.

## Desired behavior

After a valid Done-to-Todo Pomodoro reopen succeeds:

- Remove every `🍅` prefix associated with a block-link occurrence in that Pomodoro's descendant sub-bullet block,
  including live, embedded, retired/struck, duplicate, and non-reopenable-target links.
- Leave the block links themselves and all aliases, embed/strike state, surrounding prose, indentation, line endings,
  and cursor position unchanged except for restoration already performed by the reopen workflow.
- Do not change markers under later Pomodoros, outside the selected Pomodoro's sub-bullet range, or inside fenced code
  blocks. Do not remove unrelated tomato characters that are not marker prefixes for block links.
- Preserve the current root-only task reopen policy and best-effort handling of stale, missing, or unreadable targets.
- Do not clear markers when the active line is not an eligible completed Pomodoro or the Done-to-Todo transition does
  not occur.

## Implementation plan

1. In `plugins/task-status-cycler/main.js`, add a small Pomodoro-reopen edit/normalization path that uses the existing
   Pomodoro section/range and marker-rewrite primitives to strip block-link marker prefixes only from the selected
   completed Pomodoro's sub-bullet range. Make the rewrite fence-aware and derive edits from the current editor contents
   after asynchronous target reopening/reference restoration, so it composes safely with restored local links.

2. Integrate the local marker edits with `reopenActivePomodoroTask`'s successful Done-to-Todo transition. Keep the
   current ordering and semantics for resolving/deduplicating direct roots, reopening only eligible roots, restoring the
   Pomodoro's own retired references, preserving task metadata, and keeping the cursor stable. Ensure the local
   normalization covers all block-link occurrences, rather than only links whose targets changed status.

3. Extend `scripts/test-task-status-cycler.cjs` with focused coverage for the runtime behavior. Update the existing
   completed-Pomodoro reopen fixture to assert that markers are removed from embedded, plain, duplicate,
   struck/restored, already-open, and otherwise non-reopenable block links in the selected Pomodoro, while a later
   Pomodoro, fenced content, unrelated tomatoes, layout, and cursor remain unchanged. Retain or add an assertion showing
   that generic vault-wide reference restoration still preserves historical Pomodoro markers outside an explicit
   Pomodoro reopen.

4. Release the behavior as Task Status Cycler `1.3.4`: update `plugins/task-status-cycler/manifest.json` and the Task
   Status Cycler version/summary in `README.md` so the catalog describes marker reset on Pomodoro reopen.

5. Verify the change with the focused Task Status Cycler test file, the full repository test suite, and manifest
   validation. Open the required numbered linked checkout for `bob-plugins`, then deploy only `task-status-cycler` with
   `bob plugins sync --no-pull --plugin task-status-cycler --repo <the path printed by sase workspace open>` and confirm
   the deployed managed files are byte-identical to the source checkout without syncing unrelated plugins.

## Expected files

- `plugins/task-status-cycler/main.js`
- `scripts/test-task-status-cycler.cjs`
- `plugins/task-status-cycler/manifest.json`
- `README.md`
