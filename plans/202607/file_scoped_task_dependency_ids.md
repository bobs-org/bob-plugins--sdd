---
create_time: 2026-07-11 16:38:12
status: done
prompt: .sase/sdd/prompts/202607/file_scoped_task_dependency_ids.md
tier: tale
---

# Plan: File-scoped Obsidian block IDs with path-qualified task dependency IDs

## Goal

Make Obsidian block IDs unique only within their containing note while preserving correct vault-wide Obsidian Tasks
dependency semantics. A task dependency's navigation identity remains the resolved note path plus trailing block ID,
while the Tasks-facing `[id::]` and `[dependsOn::]` values become a deterministic, vault-wide identifier:

```text
<parent_file_path>__<block_id>
```

For example, a dependency task in `projects/Shared.md` with trailing block ID `^review` becomes:

```markdown
- [ ] #task Review the design [id:: projects__Shared__review] ^review
```

and every task that depends on it, whether in the same note or another note, uses:

```markdown
[dependsOn:: projects__Shared__review]
```

Its navigation bullet remains an ordinary Obsidian block link, such as `![[projects/Shared#^review]]` from another note
or `![[#^review]]` within `projects/Shared.md`.

## Audit result

The current implementation does **not** provide this guarantee:

- `bob-navigation-hotkeys` correctly resolves external transclusions as `note#^block-id`, but its cross-file toggle
  writes the target's bare block ID to `[id::]` and the same bare value to the parent's `[dependsOn::]`.
- The existing cross-file runtime test explicitly expects `[id:: target]` and `[dependsOn:: target]`, so it verifies the
  behavior that needs to change.
- `task-status-cycler` rewrites Tasks-generated IDs to bare block IDs and propagates that bare mapping across the vault.
- Obsidian Tasks and the native Dataview/Tasks parity implementation match dependencies vault-wide by equality of
  `[dependsOn::]` and `[id::]`. Two files can therefore collide if they reuse a block ID.
- The dependency-bullet migration's fallback index is keyed by bare block ID, so duplicate block IDs in different notes
  cannot be resolved safely.
- `bob mark-next-tasks` is already path-safe: its graph nodes are `(resolved file path, block ID)`. Its
  property-independent traversal does not need redesign, only a regression test.

The live vault currently contains migrated same-note dependency bullets but no cross-note dependency bullet that proves
the Tasks-facing identity behavior. The feature must therefore be corrected before cross-note dependencies are relied
on.

## Canonical identity contract

Introduce one small, explicitly tested identity function in each plugin/runtime that needs it:

```text
dependency_id(vault_relative_markdown_path, block_id)
```

The function will:

1. Normalize the target note to its vault-relative path with forward slashes.
2. Remove the final `.md` extension.
3. Replace every `/` with `__`.
4. Append `__` and the target task's unchanged trailing block ID.

Examples:

| Target                       | Canonical Tasks ID          |
| ---------------------------- | --------------------------- |
| `cash.md#^unemployment`      | `cash__unemployment`        |
| `projects/Shared.md#^review` | `projects__Shared__review`  |
| `done/team/Archive.md#^ship` | `done__team__Archive__ship` |

Preserve path casing rather than silently lowercasing it. The block token itself remains `^review`, not
`^projects__Shared__review`; only `[id::]` and `[dependsOn::]` are qualified.

Use the qualified form for **all** dependencies, including same-note dependencies. A target can have only one Tasks ID,
so allowing a bare same-note ID and a qualified cross-note ID would become inconsistent as soon as both local and remote
tasks depend on the same target.

Before writing, validate that the computed value is accepted by the Tasks ID grammar. Because the requested `__`
encoding is not injective if a filename itself contains `__`, build an identity index and treat any resulting collision
as an error: report the conflicting paths and make no ambiguous rewrite. Likewise, report unsupported path characters
rather than inventing an undocumented escaping scheme.

## Repositories and data involved

### `bob-plugins` linked repository

Access it through the mandated `sase workspace open -p bob-plugins ...` workflow.

- `plugins/bob-navigation-hotkeys/main.js`
- `plugins/task-status-cycler/main.js`
- `plugins/block-id-prompt/main.js`
- `scripts/test-navigation-hotkeys.cjs`
- `scripts/test-task-status-cycler.cjs`
- a focused block-id-prompt test file if needed
- a new idempotent task-dependency identity migration under `scripts/`
- relevant manifests and README entries

### `bob-cli`

- `src/native/collect_done.rs` for task moves and block-ID collision renames
- native Dataview/Tasks dependency fixtures and parity tests
- `mark-next-tasks` fixtures/tests for duplicate block IDs in separate notes
- user-facing documentation where the dependency identity convention is described

### Live vault

- `~/bob/`, migrated only after a dry run and explicit diff inspection
- do not commit vault changes without Bryan's approval

## Design

### 1. Centralize path-qualified identity behavior in plugin helpers

Add pure helpers for canonical path normalization, ID construction, validation, and exact comma-list replacement. Export
them through the existing test helper surfaces. Keep the implementations aligned across standalone Obsidian plugins; do
not introduce a runtime import between plugin directories, since each deployed plugin must remain self-contained.

Represent a resolved dependency internally as an object containing at least:

- target vault-relative path;
- target block ID;
- canonical Tasks dependency ID;
- any legacy/current target ID observed before normalization.

Do not use a bare block ID as the internal key for a vault-wide map. Use `(path, block ID)` for navigation identity and
the computed qualified ID for Tasks identity.

### 2. Update `bob-navigation-hotkeys` authoring and toggle flows

Change every flow that creates or removes task dependencies:

- The Ctrl+Shift+P local task picker computes the target ID from the active note path and selected task block ID. It
  writes that value to the target `[id::]` and parent `[dependsOn::]`, even though the navigation bullet is same-note.
- Prompting for a missing block ID still chooses only the local `^block-id`; after confirmation, the plugin derives the
  qualified `[id::]` from the file path instead of copying the bare block ID.
- The `!` toggle resolves same-note and cross-note transclusions to a concrete target file and task before computing the
  dependency ID. Adding `!` ensures the target carries the canonical `[id::]` and the parent carries that exact
  `[dependsOn::]`; removing `!` removes the canonical value and any proven legacy value for that same resolved target.
- If normalizing a target changes an existing ID, rewrite every proven `[dependsOn:: old]` reference to that target, not
  merely the current parent. Preserve list order, unrelated IDs, whitespace style where practical, and empty-field
  removal behavior.
- Apply external target edits before source edits and abort the source dependency addition if the target cannot be
  updated. Keep editor-buffer guards and per-file transactions so stale buffers are not overwritten.

Rework managed dependency-bullet recognition so a qualified property value can still own a bullet whose fragment is a
bare block ID. Recognition must resolve the bullet's note target and compare its canonical `(path, block ID)` identity;
it must not compare `dependsOn` text directly to the fragment. This retains the safety rule that unrelated `#^ref`
transclusions are never deleted.

### 3. Replace the `task-status-cycler` bare-ID normalizer

The post-Tasks normalizer remains responsible for changes written by the stock Obsidian Tasks auto-suggest, but its
mapping changes from `generated_id -> block_id` to `old_id -> canonical_path_qualified_id`.

- When a modified target already has a trailing block ID, compute the canonical ID from the target file path and block
  ID and replace a Tasks-generated or legacy dependency ID accordingly.
- When Tasks generates an ID for a target with no block ID, append that generated value as the file-local block ID, then
  write the path-qualified `[id::]`. This supplies the block identity required by the convention without inventing a
  second random value.
- When a dependency points to a pre-existing custom ID, resolve it only when it identifies one target unambiguously;
  ensure that target has a block ID, then canonicalize both sides. Never guess among duplicate legacy IDs.
- Propagate each exact old-to-new mapping to **all** matching dependency properties across all Markdown files. The
  current early stop after the first changed file must be removed.
- Debounce and no-op checks must still prevent modify-event loops.

Add a vault rename handler because the canonical ID contains the note path. On a Markdown rename/move, rewrite IDs in
the renamed note from the old path prefix to the new path prefix, then rewrite all exact corresponding `dependsOn`
values vault-wide. Preflight collisions and show an actionable Notice if reconciliation cannot complete. Obsidian's own
link-update behavior remains responsible for wiki-link path changes; this handler owns only dependency metadata.

### 4. Align `block-id-prompt` dependency state

Keep its picker file-local, but make its dependency lookup prefer the explicit `[id::]` exclusively when present; only
fall back to a trailing block ID for a legacy task that truly has no ID field. This prevents a stale bare
`[dependsOn:: review]` from appearing valid merely because the current note has `^review` alongside the canonical
`[id:: projects__Shared__review]`.

Add regression coverage for qualified IDs, unresolved stale bare IDs, and two files reusing the same block fragment.

### 5. Add an idempotent vault migration

Create a dry-run-by-default migration script in `bob-plugins` that updates dependency metadata and reconciles the
already-migrated navigation bullets. Reuse exported plugin helpers so runtime and migration serialize the same format.

Build a vault-wide index by `(file path, block ID)`, existing `[id::]`, and computed canonical ID. Resolve each
dependency edge in this order:

1. An exact managed transclusion target (`note#^block-id`) beneath that parent task.
2. A same-note task whose existing ID or block ID matches the legacy value.
3. A unique vault-wide task whose existing ID matches the legacy value.
4. Otherwise, report the edge as unresolved or ambiguous and leave it unchanged.

For every resolved edge:

- set the target task's `[id::]` to the canonical path-qualified value;
- replace the corresponding parent `[dependsOn::]` token with that same value;
- keep the navigation bullet fragment/path unchanged except for normal canonical link formatting already owned by the
  dependency-bullet migration;
- update every dependent edge that resolves to the same target;
- include `done/`, but skip `.git`, `.obsidian`, `_generated`, and `_templates` as before.

Perform a complete collision and ambiguity preflight before `--write`. Never choose the first task for a duplicate bare
block ID or duplicate legacy ID. Print per-file changes plus totals for target IDs, dependency values, unresolved edges,
ambiguous edges, and encoding collisions. A second dry run after writing must report zero changes.

### 6. Preserve identities when `bob move-done-tasks` relocates tasks

`move-done-tasks` changes both the containing file and, when an archive collision exists, sometimes the block ID. Extend
its existing moved-block target map so each moved task also has an old and new canonical dependency ID:

```text
(source path, original block ID) -> (archive path, final block ID)
```

During the existing all-vault repair plan:

- rewrite the moved task's `[id::]` to the canonical archive-path/final-block value;
- rewrite exact `[dependsOn:: old_canonical_id]` tokens in all planned source, archive, and link-only repair files;
- combine metadata and link repairs into the same preview/write plan and include metadata counts in human output;
- preserve dry-run, atomic-write, git preparation, and idempotency behavior.

Do not rewrite arbitrary prose occurrences or unresolved legacy IDs. Add fixtures for a normal move, a move into a
subdirectory archive, a destination block-ID collision, multiple dependents in separate notes, and a no-op rerun.

### 7. Keep downstream query and graph semantics honest

The native Dataview/Tasks implementation already compares exact `[id::]` and `[dependsOn::]` values vault-wide, so its
algorithm should not special-case paths. Add parity fixtures proving that:

- two files may both contain `^review`;
- their IDs (`alpha__review` and `beta__review`) remain distinct;
- a dependent task blocks only on the qualified target it names;
- underscores in qualified IDs parse and render without loss.

`bob mark-next-tasks` continues to ignore these properties and traverse resolved `(path, block ID)` links. Add an
integration fixture with the same `^dep` fragment in two different notes and assert that the explicit transclusion path
selects only the intended dependency. No production algorithm change is expected there.

Document the two-layer identity model: block links are file-scoped path-plus-fragment identities, while Tasks metadata
uses the deterministic path-qualified string.

## Testing and verification

### `bob-plugins`

Run `npm test` and `npm run validate`. Add focused tests for:

- root and nested note path encoding;
- same-note and cross-note authoring producing qualified IDs;
- target `[id::]`, parent `[dependsOn::]`, and bare navigation fragment staying consistent;
- a target shared by local and remote dependents using one qualified ID everywhere;
- two notes safely reusing the same block ID;
- toggle add/remove, including exact legacy cleanup and unrelated-transclusion protection;
- Tasks-generated ID normalization with and without an existing block ID;
- propagation to multiple dependent files;
- note rename/move rewriting both target IDs and all dependents;
- collision/ambiguity refusal and idempotency;
- migration dry-run/write/no-op behavior.

Deploy the changed plugins with `bob plugins sync` only after tests and manifest validation pass.

### `bob-cli`

Run the full `just` pipeline. Cover `move-done-tasks` metadata repair and the Dataview/mark-next duplicate-block-ID
fixtures described above. Confirm existing JSON schemas remain stable unless a new move metadata count is deliberately
added and documented.

### Live vault rollout

1. Run the new migration in dry-run mode and inspect every unresolved, ambiguous, or colliding entry.
2. Compare the proposed resolution set with the known dependency properties and their transcluded bullets.
3. Run `--write` only after the preflight is clean or all retained exceptions are understood.
4. Inspect `git -C ~/bob diff`, especially nested-note paths, multi-dependency fields, `done/`, and tasks whose ID
   previously differed from their block ID.
5. Run the migration dry-run again and require zero proposed changes.
6. Run native Dataview/Tasks parity and `bob mark-next-tasks --dry-run --format json` against the live vault.
7. In Obsidian, verify a same-note dependency, a cross-note dependency, a duplicate block ID in two notes, and a note
   move/rename.
8. Leave vault changes uncommitted for Bryan's review.

## Sequencing

1. Add canonical identity helpers and tests.
2. Update `bob-navigation-hotkeys`, `task-status-cycler`, and `block-id-prompt` flows.
3. Implement and test the migration; deploy plugins; migrate and inspect the live vault.
4. Update `move-done-tasks` and downstream parity/graph regression tests.
5. Run full repository checks and live-vault dry-run verification.

## Risks and safeguards

- **Path-derived IDs change on rename:** handle Obsidian rename events and CLI archival moves explicitly.
- **Encoding collisions:** preflight and fail; never silently choose between `a/b` and a literal `a__b` path.
- **Legacy ambiguity:** use resolved navigation links and same-note context first; never use first-match vault-wide
  logic.
- **Partial multi-file edits:** update/verify the target before adding a source dependency, retain stale-buffer guards,
  and make migrations preflight-complete and idempotent.
- **Unrelated transclusions:** managed bullets still require resolution to a task and membership in the parent's
  resolved dependency set.
- **File-local block IDs:** uniqueness prompts and archive deduplication remain scoped to the concrete target file, not
  the whole vault.
- **Stock Tasks compatibility:** validate the underscore-based qualified ID grammar against the installed/supported
  Tasks version and preserve exact equality semantics rather than teaching downstream queries a second resolver.
