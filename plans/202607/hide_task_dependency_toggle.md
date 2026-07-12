---
create_time: 2026-07-12 19:11:02
status: done
prompt: 202607/prompts/hide_task_dependency_toggle.md
tier: tale
---

# Plan: Exclude hidden tasks from `!` dependency linking

## Goal

Change Bob Navigation Hotkeys so Obsidian's `!` transclusion keymap adds a linked task to its parent task's dependency
metadata only when the linked target task does not carry a standalone `#hide` tag. Preserve `!` as a general
link/transclusion toggle even when dependency synchronization is intentionally skipped.

## Current behavior and scope

Bob Navigation Hotkeys toggles ordinary links between `[[...]]` and `![[...]]`. For a managed child block link beneath a
real `#task`, the same operation also treats the linked task as a Tasks dependency: it updates the parent task's
`[dependsOn:: ...]`, gives the target a canonical `[id:: ...]` when adding the dependency, and performs the same
synchronization for same-note, cross-note, and counted Vim toggles.

The change belongs in the `bob-plugins` source-of-truth repository, specifically the `bob-navigation-hotkeys` plugin. It
should affect only the dependency side effect of the `!` keymap. It should not prevent a hidden task's link from
becoming visually embedded, and it should not broaden into changing the explicit dependency property picker or unrelated
dependency-navigation/migration behavior.

## Behavioral decisions

- Evaluate `#hide` on the linked target task, not on the parent task containing the child link.
- Recognize only a whole `#hide` task tag using the plugin's existing task-tag boundary semantics. Text such as
  `#hidden` must not suppress dependency linking, while punctuation-delimited `#hide` must.
- On an add transition (`[[target]]` to `![[target]]`), a hidden target is ineligible for dependency synchronization.
  Toggle the embed marker, but do not add `[dependsOn:: ...]`, add or rewrite the target's `[id:: ...]`, propagate
  legacy dependency identities, or write an external target note.
- On a remove transition (`![[target]]` to `[[target]]`), continue removing the target from the parent's dependency list
  even if the target currently has `#hide`. This allows the keymap to clean up an existing dependency after a task
  becomes hidden. As today, retain the target's existing `[id:: ...]` field.
- Apply these rules consistently to same-file links, cross-file links, and counted/multi-line `!` operations. Other
  links on the same line or in the same counted operation must continue toggling and synchronizing independently.
- Preserve all existing stale-editor, invalid-parent, invalid-target, unsupported-path, and external-write safety
  checks.

## Implementation plan

1. In `plugins/bob-navigation-hotkeys/main.js`, introduce or expose a focused whole-tag predicate based on the existing
   `getWholeTaskTagSpans(..., "#hide")` logic, so dependency-toggle eligibility uses the same syntax already used by
   project scheduling and does not duplicate a looser substring or regular-expression check.

2. Apply target visibility eligibility at the dependency action boundary rather than in general task lookup:
   - Keep locating hidden task targets by block ID so an embed-removal transition can still unlink stale dependency
     metadata.
   - In the pure same-file planner, short-circuit only the dependency-add path when the resolved target task has
     `#hide`; return the visually toggled content as an unqualified/plain transclusion change with a specific reason and
     without target-ID or dependency edits.
   - In the runtime same-file/cross-file planner, inspect the resolved target snapshot before scheduling canonical-ID
     normalization, external `vault.process` writes, legacy-ID propagation, or parent `[dependsOn:: ...]` actions. Skip
     those add-side effects for hidden targets while leaving the already-planned embed-marker edit intact.
   - Leave removal actions eligible for hidden targets, ensuring existing canonical, legacy, and block-ID dependency
     values are still removed through the current cleanup path.

3. Extend `scripts/test-navigation-hotkeys.cjs` with focused regression coverage:
   - Same-file hidden targets visually embed without gaining `[id:: ...]` and without changing the parent's
     `[dependsOn:: ...]`.
   - A near-match such as `#hidden` remains dependency-eligible, and standalone `#hide` boundary forms are excluded.
   - A previously linked target that now has `#hide` can be unembedded and removed from `[dependsOn:: ...]` without
     deleting its existing `[id:: ...]`.
   - Cross-file hidden targets cause no external write on add, while the source link still embeds; visible cross-file
     targets retain the existing synchronization behavior.
   - Counted/mixed toggles handle hidden and visible targets independently, proving one hidden target does not suppress
     valid dependency actions elsewhere in the batch.

4. Document and release the user-visible plugin change:
   - Update the Bob Navigation Hotkeys description/version entry in `README.md` to mention that `!` synchronizes only
     visible task dependencies.
   - Increment the patch version in `plugins/bob-navigation-hotkeys/manifest.json` and keep the README version table
     aligned.

5. Validate and deploy:
   - Run the focused navigation-hotkeys test file, then the full `npm test` suite.
   - Run `npm run validate` to check manifests and plugin syntax.
   - Run `bob plugins sync -p bob-navigation-hotkeys` to deploy the source-of-truth plugin change to the Bob vault, as
     required by the repository workflow.

## Acceptance criteria

- Pressing `!` on a managed link to a visible `#task` preserves current embed and dependency synchronization behavior.
- Pressing `!` to embed a managed link to a target task tagged with standalone `#hide` changes only the link's embed
  marker; neither note receives new dependency metadata.
- Pressing `!` to unembed that link removes any existing dependency reference from the parent, even when the target is
  hidden.
- Same-file, cross-file, and counted toggles all follow the same rule without weakening existing concurrency and
  validation guards.
- `#hidden` and other non-whole-tag text do not count as `#hide`.
- All tests and manifest validation pass, and the updated plugin is synchronized to the vault.
