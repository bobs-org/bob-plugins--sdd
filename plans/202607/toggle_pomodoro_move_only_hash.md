---
create_time: 2026-07-14 07:59:54
status: done
prompt: 202607/prompts/toggle_pomodoro_move_only_hash.md
tier: tale
---

# Plan: Toggle Pomodoro Move-Only `#` Directives

## Goal and user-visible behavior

Change the existing normal-mode `#` mapping in the `task-status-cycler` plugin from add-only marking to a true toggle
for strict Pomodoro move-only directives.

- Preserve the current range semantics: bare `#` considers only the cursor line, while explicit `N#` considers the
  cursor line plus the next `N` physical lines, clamped to the current open Pomodoro's sub-bullet block.
- On every eligible line in that range, invert the strict move-only state independently:
  - `- [[...#^task]]` becomes `- [[...#^task]]#` after its linked block resolves to a real Obsidian Tasks-plugin task.
  - `- [[...#^task]]#` becomes `- [[...#^task]]` without requiring the linked target to resolve or remain a task.
- Mixed counted ranges therefore toggle each eligible link according to its own current state. Ineligible physical lines
  remain unchanged and still count toward the range, matching the existing `N#` behavior.
- Keep the existing strict syntax boundary. Only a single directive immediately after the closing `]]` and before
  trailing horizontal whitespace is removable. Similar text such as `]] #`, `]]##`, `]]# trailing prose`, embedded or
  struck links, prose/multiple-link bullets, malformed links, and fenced examples must remain untouched.
- Preserve aliases, paths, indentation, line endings, trailing whitespace, unrelated text, and the original cursor
  position. The action remains unavailable outside the sub-bullet block of an open top-level Pomodoro.

Removal is intentionally local and does not revalidate the destination. A task can be renamed, deleted, or cease being a
Tasks task after it was marked; requiring resolution on toggle-off would strand a stale move-only directive that the
user is explicitly trying to remove. Additions retain the current target validation so the keymap cannot create new
move-only directives for stale links or non-task blocks.

## Implementation approach

1. Generalize the pure move-only range planner in `plugins/task-status-cycler/main.js` from add-only edits into toggle
   edits. Reuse the existing strict move-only parser's clean destination text for removals and the strict bare-link
   parser for additions. Represent each planned edit with enough intent to let the runtime distinguish local removals
   from additions that need asynchronous target validation, while retaining the current eligibility, range clamping, and
   source-line snapshots used for safe application.

2. Update the asynchronous editor operation so it validates only planned additions through the existing same-note and
   cross-note block resolver, still requiring a proper `#task` checkbox target. Accept planned removals immediately,
   tolerate validation failure on one addition without blocking removals or other valid additions in the same counted
   range, and apply all accepted edits only when their live source lines still match the planner snapshots. Continue to
   restore the original cursor and contain asynchronous failures at the Vim action boundary.

3. Align internal action/helper naming and comments with toggle semantics where doing so improves clarity, while
   preserving the external normal-mode `#` mapping and its bare-versus-explicit count handling. No new setting, command,
   or key binding is needed.

4. Expand `scripts/test-task-status-cycler.cjs` to cover the behavior change and replace the former add-only idempotence
   expectation with toggle/round-trip expectations. Include focused cases for:
   - bare `#` adding an unmarked directive and removing an existing strict directive;
   - a second keypress restoring an eligible, resolvable line exactly, including alias and trailing whitespace;
   - `1#` and larger mixed ranges that remove marked links, add valid unmarked links, skip prose/ineligible lines, and
     stop at the owning Pomodoro boundary;
   - removal from stale, missing, unreadable, or no-longer-task targets without resolver approval;
   - continued rejection of additions for stale and non-task targets, with independent best-effort results when a
     counted range contains removals, valid additions, and invalid additions together;
   - strict non-matches such as spaced, doubled, embedded, struck, mixed-content, malformed, fenced, outside-section,
     and completed-Pomodoro cases;
   - mapping registration, explicit-count interpretation, live-line race protection, cursor preservation, and contained
     asynchronous failures.

5. Release the behavior refinement from the `bob-plugins` source of truth as a patch update to `task-status-cycler`
   (from 1.5.0 to 1.5.1). Update the plugin manifest description and repository README entry so they describe `#`/`N#`
   as toggling move-only directives rather than only marking them, while keeping the existing Ctrl+Enter carry-forward
   behavior clear.

## Verification and deployment

1. Run the focused task-status-cycler test file, then the complete `npm test` suite and `npm run validate` from the
   linked `bob-plugins` checkout.
2. Review the final diff to ensure it is limited to the plugin implementation, focused tests, manifest version and
   description, and README release metadata.
3. Preview deployment with `bob plugins sync --no-pull --dry-run -p task-status-cycler -r .`, then deploy with
   `bob plugins sync --no-pull -p task-status-cycler -r .`. Verify the deployed `main.js` and manifest match the
   validated source and that a second dry run is a no-op.
4. Manually verify in Obsidian that bare `#` toggles a marked link off and back on without moving the cursor, and that
   `1#` over one marked and one unmarked valid task link swaps both states. Complete the Pomodoro with Ctrl+Enter to
   confirm only links left marked at completion use the existing move-only carry-forward path.
