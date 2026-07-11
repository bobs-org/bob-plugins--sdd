---
create_time: 2026-07-11 17:49:22
status: done
tier: tale
---
# Plan: Restrict dependency semantics to valid Obsidian tasks

## Problem statement

The dependency-aware transclusion behavior added to `bob-navigation-hotkeys` currently uses task-shaped Markdown as its
eligibility test in several places:

- the Set bullet property picker allows a `local_task_id`/`dependsOn` property on any list item;
- the `!` transclusion toggle accepts any checkbox as the parent and any checkbox with the requested block ID as the
  dependency target; and
- the dependency-bullet rewrite/migration helpers accept any bullet carrying `[dependsOn:: ...]` and index targets with
  a broad checkbox regex.

A Pomodoro ledger entry such as `- [ ] (**1535-1705** [t:: 90m])` is therefore mistaken for a dependency-bearing task
when `!` is used on one of its child task links. The link should still toggle between `[[...]]` and `![[...]]`, but the
Pomodoro line must not gain `[dependsOn:: ...]`, and the linked target must not be mutated as a side effect of an
ineligible parent.

The vault's Obsidian Tasks global filter is `#task`. For this feature, a **valid Obsidian task** is therefore a genuine
Markdown checkbox list item whose body contains a standalone `#task` token, outside frontmatter and fenced code. Task
validity is independent of status: completed and custom-status `#task` lines remain real tasks, while “open task” stays
a narrower predicate used only where the UI intentionally lists open dependency targets.

## Scope and repositories

The implementation belongs in the linked `bob-plugins` repository, primarily:

- `plugins/bob-navigation-hotkeys/main.js`
- `scripts/test-navigation-hotkeys.cjs`
- `scripts/migrate-dependency-bullets.mjs`
- `plugins/bob-navigation-hotkeys/manifest.json`

Implementers must open the linked repository with the required `sase workspace open -p bob-plugins ...` workflow and use
the returned path.

No `bob-cli` behavior change is planned. `mark-next-tasks` already builds dependency edges only from tasks parsed with
the configured Tasks global filter (currently `#task`), and Pomodoro entries are only direct-reference containers, not
dependency graph nodes.

## Current-state findings

1. `OBSIDIAN_TASK_LINE_RE` only proves that a line is a checkbox; it does not prove that Obsidian Tasks recognizes it.
   `isOpenObsidianTaskLine` adds the missing standalone-`#task` check, but also applies open-status filtering and is
   therefore too narrow to serve as the general validity predicate.
2. `findDependencyToggleParent` and `findTaskLineByBlockId` currently use only `OBSIDIAN_TASK_LINE_RE`. Both same-file
   helper logic and the runtime cross-file flow inherit this gap.
3. `applyDependencyAwareTransclusionChanges` discovers and edits a target before applying the parent `[dependsOn::]`
   edit. Eligibility must be established before either side is mutated so an invalid parent cannot still add `[id::]` to
   a target.
4. The property picker initially guards only with `isBulletLine`, and its single- and multi-select dependency paths
   revalidate only that broad condition. This is a second independent route for adding `dependsOn` to non-tasks.
5. The one-off dependency-bullet migration intentionally handled plain bullets under the former design. Its shared
   transform and target index must now adopt the corrected task boundary so future dry runs or the in-editor rewrite
   command cannot expand dependency navigation on non-tasks.
6. A read-only vault audit found no current `[dependsOn::]` properties in daily-note Pomodoro sections. It did find
   eight pre-existing non-`#task` bullets with dependency metadata elsewhere. Those may be historical intentional data,
   so this fix must report/ignore them rather than silently deleting user content.

## Design

### 1. Introduce one canonical valid-task predicate

Add a line-level `isObsidianTaskLine` helper that requires:

- a valid Markdown checkbox list-item shape supported by the existing parser; and
- a standalone `#task` token using the existing tag-boundary regex.

Refactor `isOpenObsidianTaskLine` to build on the new predicate and then apply `OPEN_OBSIDIAN_TASK_STATUSES`. Add or
reuse a content-aware scanner/context check for operations that process a whole note so task-shaped examples in YAML
frontmatter or fenced code are not eligible.

Keep lifecycle concerns separate:

- `isObsidianTaskLine`: valid Tasks-plugin task, any one-character status;
- `isOpenObsidianTaskLine`: valid task in one of the statuses offered by the dependency picker; and
- `isPomodoroNavigationTaskLine`: ledger navigation only, never dependency eligibility unless the line independently
  carries standalone `#task` and therefore genuinely is a Tasks task.

Export the new helper for focused unit tests.

### 2. Guard the Set bullet property dependency flow

Restrict properties whose configured value source is `local_task_id` to a valid `#task` parent. This should be based on
the property type rather than hard-coding the name `dependsOn`, so all dependency-style configuration follows the same
invariant.

- On a non-task bullet, do not offer an undefined `local_task_id` property as something that can be added.
- If an invalid line already carries such a property, keep a cleanup path available (for example, surface it as
  removable but do not allow entering the task-selection stage) so the UI does not trap bad historical metadata.
- Before every single-select, prompted-ID, and batch dependency write, re-read the live cursor line and validate both
  task shape and note context. Abort with a clear notice such as “Dependencies can only be set on #task checkboxes.”
- Perform this guard before changing a selected target's trailing block ID or `[id::]`, preventing partial mutation when
  the parent is invalid or became stale while the modal was open.
- Keep non-dependency bullet properties available on ordinary bullets exactly as today.

Use a shared validation helper across modal entry, single selection, block-ID confirmation, and batch execution instead
of relying on UI filtering as the only safeguard.

### 3. Guard dependency-aware `!` side effects while preserving plain toggling

Update both the pure same-file planner and `applyDependencyAwareTransclusionChanges` so dependency synchronization
qualifies only when all of the following hold:

1. the changed line is still a sole block-link child bullet;
2. its nearest direct parent list item is a valid `#task` line in real note content;
3. the link resolves to a valid `#task` line carrying the referenced block ID; and
4. the source/target snapshots remain unchanged when the edit is applied.

If any condition fails—including when the parent is a Pomodoro ledger checkbox—the ordinary transclusion text edit must
still succeed, but there must be no `[dependsOn::]`, `[id::]`, block-ID normalization, or vault-wide dependency-ID
rewrite side effect.

Apply the same rule to same-note and cross-note targets and to every line in a counted `N!` range. Build the action list
only after parent and target validation; external `vault.process` edits must be scheduled only for already-qualified
actions.

Preserve existing valid-task behavior: adding `!` adds/deduplicates the canonical dependency ID and ensures the target
identity; removing `!` removes the matching dependency value and drops an empty property.

### 4. Align dependency navigation rewrite and migration tooling

Make note-wide dependency navigation transformations operate only on valid `#task` parents outside frontmatter/fences.
Likewise, make the migration target index resolve `[id::]` and `^block-id` identities only from valid `#task` lines, not
arbitrary checkboxes or block-bearing prose.

For non-task lines that already contain `[dependsOn::]`:

- leave the property and any child content unchanged;
- exclude them from changed-task/dependency-bullet totals; and
- report a skipped-non-task count (and file/line details where practical) in migration dry-run output.

Do not add an automatic vault cleanup that deletes the eight currently observed non-task properties. They predate this
fix and cannot safely be classified as accidental. After deployment, run a read-only audit of Pomodoro sections; only
propose a separate targeted cleanup if invalid Pomodoro metadata actually exists at that time.

The in-editor “Rewrite dependency navigation links” command should use the same eligible-task transform, so it cannot
reintroduce navigation bullets beneath Pomodoros or plain backlog bullets.

### 5. Version, deploy, and document the invariant in code

Bump `bob-navigation-hotkeys` from `1.12.0` to `1.12.1` as a behavior-correcting release. Update nearby comments so
future changes do not collapse “checkbox,” “Pomodoro navigation target,” “valid Obsidian task,” and “open Obsidian task”
into one concept.

Deploy only after tests pass using `bob plugins sync -r <bob-plugins-workspace-root>`. Do not edit the deployed copy in
the vault directly.

## Testing and verification

Extend `scripts/test-navigation-hotkeys.cjs` with regressions covering:

- valid-task classification: unordered/ordered/blockquote `#task` checkboxes, open/done/custom statuses, standalone tag
  boundaries, and rejection of Pomodoro checkboxes, plain checkboxes, `#taskish`, plain bullets, frontmatter, and fenced
  examples;
- property picker items: `local_task_id` is addable on valid tasks, unavailable for addition on Pomodoros/plain bullets,
  and pre-existing invalid metadata remains removable but cannot open the dependency-selection flow;
- defense-in-depth write guards: single selection, prompted block ID, and batch execution perform no parent or target
  mutation after the parent becomes invalid/stale;
- same-file `!`: a valid parent and valid target still synchronize, a Pomodoro parent only gains/removes `!`, and an
  invalid target receives neither `[id::]` nor a dependency edge;
- cross-file runtime `!`: valid task-to-task synchronization still works; invalid parent and invalid target cases leave
  the external file untouched while applying the source link toggle;
- counted ranges mixing valid task children, Pomodoro children, and unrelated links: every link toggles, but only the
  valid task-to-task line gets dependency side effects;
- transform/migration: valid task parents are rewritten idempotently; plain bullets, Pomodoro entries, fenced examples,
  and non-task targets are skipped and reported; unrelated child transclusions remain untouched.

Run:

1. `npm test` in `bob-plugins`;
2. `npm run validate` for plugin manifests;
3. the dependency migration in dry-run mode against `~/bob` and verify it proposes no non-task rewrites;
4. `bob plugins sync -r <bob-plugins-workspace-root>`;
5. a manual Obsidian smoke test in a disposable/current Pomodoro entry: toggle a child task link with `!`, confirm the
   embed marker changes, and confirm the Pomodoro line and linked task metadata do not change;
6. a corresponding valid `#task` smoke test confirming dependency add/remove and target identity behavior still work;
7. `bob mark-next-tasks --dry-run --format json` to confirm the CLI still treats Pomodoro child links as direct task
   references and valid task transclusions as dependency edges.

## Sequencing

1. Add and test the canonical valid-task/context helpers.
2. Apply the parent guard to all property-picker dependency paths before any target mutation.
3. Apply parent/target guards to same-file, cross-file, and counted transclusion side effects.
4. Restrict note-wide rewrite/migration behavior and add skipped-non-task reporting.
5. Run plugin tests and manifest validation; bump the patch version.
6. Run the live-vault dry-run audit, deploy the plugin, and perform the two smoke tests plus CLI dry run.

## Risks and invariants

- **Ordinary `!` behavior must never be blocked.** Ineligible dependency semantics mean “toggle only,” not “reject the
  command.”
- **Validate before target mutation.** A parent guard placed only in the final `[dependsOn::]` writer is too late
  because current flows may already have added `[id::]` or a block ID to the target.
- **Do not equate valid with open.** Completed/custom-status `#task` lines remain legitimate Tasks records and existing
  dependency navigation must not be orphaned merely because status changed.
- **Do not silently clean historical metadata.** Non-task `[dependsOn::]` fields outside Pomodoros are reported and left
  untouched unless Bryan separately approves their removal.
- **Keep CLI and plugin boundaries consistent.** Both should require the configured/global `#task` filter for graph
  nodes; Pomodoro entries remain reference containers only.
- **Preserve live-vault safety.** Migration/audit is dry-run first, vault changes are reviewed, and no vault commit is
  created without explicit approval.
