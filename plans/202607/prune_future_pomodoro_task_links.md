---
create_time: 2026-07-13 08:25:18
status: done
prompt: 202607/prompts/prune_future_pomodoro_task_links.md
tier: tale
---

# Plan: Prune Future Pomodoro Task Links from the `^^` Picker

## Goal

Extend the `block-id-prompt` Obsidian plugin so that completing a task block link through the special `^^` task picker
also removes occurrences of that same task from later Pomodoros in the source daily note. The cleanup must run only when
the marker being completed is a sub-bullet of an open Bob Pomodoro. A `^^` link completed beneath a closed Pomodoro,
beneath a normal Obsidian task, or outside the `## Pomodoros` section must retain today's behavior and must not delete
anything.

The implementation belongs in the linked `bob-plugins` source-of-truth repository. No `bob-cli` command behavior needs
to change.

## Behavioral Contract

- Recognize the owning Pomodoro structurally: the source marker must be an indented list descendant of a valid,
  top-level Bob ledger entry in the current note's `## Pomodoros` section. A normal checkbox task is not a Pomodoro just
  because it has indented children.
- Treat a recognized ledger entry as open when its supported checkbox status is not completed or canceled, matching the
  existing Bob ledger distinction. Closed `[x]`, `[X]`, and `[-]` Pomodoros never trigger cleanup.
- Define "future Pomodoros" as valid open Pomodoro entries appearing after the owning Pomodoro in that same section.
  Leave the owning entry, all earlier entries, later closed history, fenced examples, and content outside the section
  untouched.
- Identify "the same task" by resolved destination Markdown-file path plus block ID, not by raw link spelling. This
  allows aliases, embeds, same-note links, and alternate valid Obsidian link paths to match while leaving the same block
  ID in a different note alone. Unresolvable links are preserved rather than guessed.
- Remove every matching Obsidian wiki block-link occurrence beneath later open Pomodoros. For a dedicated task-link
  bullet, remove the bullet as a unit (including its nested continuation block) so no orphaned child indentation is left
  behind. For mixed-content bullets, remove only the matching link token and its attributable embed/marker or exact
  strike wrapper while preserving authored text and unrelated links.
- Run the cleanup for both `^^` completion paths: tasks that already have a block ID and tasks that receive a new block
  ID through the follow-up prompt. If no future occurrence exists, completion remains a no-op beyond the existing link
  and task-status changes.
- Preserve the existing guarded-write behavior: revalidate the selected task and live source marker, build
  non-overlapping edits from a single editor snapshot, and apply the current-link completion plus all future cleanup
  edits together to the active editor. Keep the source cursor at the completed link and report the number of removed
  future occurrences in the success notice. Do not leave a partially cleaned daily note if its live buffer changed
  during the picker flow.

## Implementation

1. **Make Pomodoro ancestry explicit in `plugins/block-id-prompt/main.js`.** Refactor the current boolean sub-bullet
   detection into a source-context helper that returns the Pomodoros section, owning ledger line, checkbox status, and
   open/closed state. Retain the existing task-to-Next behavior, but give the new cleanup path its own strict
   open-Pomodoro gate so normal tasks and closed Pomodoros cannot qualify accidentally.

2. **Add a pure future-link cleanup planner.** Parse later valid open Pomodoro entry ranges, skip fenced content,
   collect supported wiki block references, resolve their note targets through an injected resolver, and compare
   resolved path plus block ID with the selected task. Produce deterministic, non-overlapping text edits and a
   removed-occurrence count. Reuse the plugin's existing block reference parsing, line-offset, and reverse-edit
   machinery where possible, while adding the list-bullet/subtree logic needed to remove dedicated scheduled-task
   bullets safely and preserve line endings.

3. **Integrate cleanup into both successful task-picker completion paths.** Once the final task identity is known, plan
   the source-note update from the live editor content and finalize the current `^^` marker and future-link removals as
   one source edit. Account for the existing destination task edit (Next promotion and/or block-ID insertion), including
   the same-file case, by planning from the post-task-edit live source snapshot. Keep existing failure notices and add
   concise success text such as `removed 2 future links` only when the cleanup changed the note.

4. **Add focused regression coverage in `scripts/test-block-id-prompt.cjs`.** Cover open versus closed owning Pomodoros;
   normal Obsidian task ancestry; later open versus current, earlier, and later closed entries; multiple duplicates;
   aliases, embeds, same-note links, alternate paths, and same-ID/different-file non-matches; mixed-content and
   dedicated bullets with descendants; fenced examples; CRLF preservation; unresolved targets; no-match idempotence; and
   both existing-ID and newly-created-ID completion flows. Export only the minimal pure helpers needed by the Node test
   harness.

5. **Release, document, validate, and deploy the plugin.** Bump the `block-id-prompt` patch version in
   `plugins/block-id-prompt/manifest.json` and update its entry in the `bob-plugins` README to mention future-Pomodoro
   de-duplication. Run the focused block-ID-prompt tests, the complete `npm test` suite, and `npm run validate`. Finally
   run `bob plugins sync -p block-id-prompt` as required by the linked repository instructions, then verify
   `bob plugins list` reports the plugin as synced.

## Acceptance Example

Starting with:

```markdown
## Pomodoros

- [x] Earlier (0900-0925)
  - [[projects/Alpha#^ship]]
- [ ] Current (0930-0955)
  - [[projects/Alpha]]^^
- [ ] Later ()
  - [[Alpha#^ship|Ship it]]
  - Keep this note
- [x] Closed history (1000-1025)
  - [[projects/Alpha#^ship]]
```

choosing task `^ship` completes the current link and removes the dedicated matching bullet under `Later`. The earlier
and later closed-history links remain untouched, as does `Keep this note`. Performing the same `^^` action beneath an
ordinary task or a closed Pomodoro removes no future links.
