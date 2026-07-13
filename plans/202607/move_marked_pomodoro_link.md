---
create_time: 2026-07-13 11:41:46
status: done
prompt: 202607/prompts/move_marked_pomodoro_link.md
tier: tale
---

# Move `#`-marked Pomodoro task links on completion

## Goal

Extend the `task-status-cycler` Obsidian plugin so that `Ctrl+Enter` treats an immediate trailing `#` on a standalone,
non-transcluded task block-link sub-bullet as a move-only carry-forward directive. For example, completing:

```markdown
- [ ] (**1110-1135** [t:: 25m])
  - [[#^gtd]]#
  - foo bar baz
```

must produce:

```markdown
- [x] (**1110-1135** [t:: 25m])
  - foo bar baz
- [ ] ()
  - [[#^gtd]]
```

The completed Pomodoro must not retain a `🍅 [[#^gtd]]` history entry, and the destination link must contain neither the
directive suffix nor a tomato marker.

## Design

1. Add a narrowly scoped parser/classification for move-only Pomodoro bullets in `plugins/task-status-cycler/main.js`.
   - Recognize a live, non-embedded task block link whose standalone list-item body ends with `]]#` (allowing only
     trailing whitespace after the directive).
   - Reuse the existing block-link parser so same-note links, note-qualified links, aliases, and valid block IDs follow
     the plugin's current rules.
   - Keep the syntax deliberately strict: prose/mixed-link bullets, embedded links, struck/retired links, fenced
     examples, a spaced `]] #`, ordinary Markdown tags, and hashes elsewhere remain ordinary content. This prevents a
     literal hash from deleting a whole bullet unexpectedly.
   - Return both the original target and a cleaned destination line with only the directive `#` removed. Continue
     exposing the target through the existing bare non-transcluded task path so the suffix does not accidentally disable
     established linked-task start behavior; the directive changes ledger placement, not target-task lifecycle behavior.

2. Integrate the new classification into Pomodoro completion planning.
   - Treat marked bullets as carry-forward content so they force creation of the fresh `- [ ] ()` Pomodoro even when
     another Pomodoro already follows.
   - Preserve source ordering when normal copied links and move-only links coexist.
   - For ordinary links, retain the existing behavior: normalize a tomato-marked historical occurrence under the
     completed Pomodoro and carry a clean copy forward.
   - For move-only links, remove the entire source sub-bullet, carry its cleaned form forward exactly once, and skip
     completed-Pomodoro tomato normalization for that removed occurrence.
   - Preserve every unrelated note bullet and all existing embedded/retired-link completion behavior.

3. Extend the editor-plan application path to support line removal safely.
   - Represent source removals explicitly rather than replacing a moved bullet with a blank line.
   - Apply insertions/removals in an order that remains correct for one or several marked bullets.
   - Account for removed lines when calculating the final placeholder/cursor line, so `Ctrl+Enter` still lands inside
     the newly created Pomodoro's `()` after the source block shrinks.

4. Add focused and end-to-end regressions in `scripts/test-task-status-cycler.cjs`.
   - Assert the exact requested transformation, including preservation of `foo bar baz`, removal of the source link
     bullet, removal of the suffix, and absence of a tomato-prefixed historical link.
   - Cover multiple marked links, aliases/path-qualified links, coexistence with ordinary copied links, stable source
     order, and a pre-existing later Pomodoro.
   - Verify strict non-matches (embedded, retired, fenced, mixed/prose, and non-immediate hashes) remain unchanged by
     move-only handling.
   - Exercise the full completion method to confirm target-task side effects remain compatible, the new Pomodoro is
     created once, and the cursor targets the correct post-removal line.
   - Retain the existing tests that prove ordinary carry-forward links still leave tomato-marked history and embedded
     task trees still complete/retire normally.

5. Release and deploy the plugin change.
   - Bump `plugins/task-status-cycler/manifest.json` from `1.3.4` to `1.4.0` because this adds user-visible marker
     syntax, and update its description if needed.
   - Update the Task Status Cycler version/summary in `README.md` to match the manifest and mention move-only Pomodoro
     carry-forward support.
   - Run `npm test` and `npm run validate` in the `bob-plugins` repository.
   - After all source changes and checks pass, run `bob plugins sync` as required to deploy the source-of-truth plugin
     into the vault.

## Expected files

- `plugins/task-status-cycler/main.js`
- `scripts/test-task-status-cycler.cjs`
- `plugins/task-status-cycler/manifest.json`
- `README.md`

No `bob-cli` command or option changes are needed.
