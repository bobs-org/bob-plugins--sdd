---
create_time: 2026-07-11 16:08:58
status: done
prompt: .sase/sdd/prompts/202607/transcluded_task_deps.md
tier: tale
---

# Plan: Transcluded per-dependency sub-bullets for Obsidian task dependencies

## Problem statement

Task dependencies in the Bob vault (`~/bob/`) are currently rendered as a single managed sub-bullet
(`- 🔗 **DEPENDS ON:** [[#^a]] • [[#^b]]`) beneath the dependent task, kept in sync with the machine-readable
`[dependsOn:: a, b]` inline property by the `bob-navigation-hotkeys` plugin. Bryan wants to replace that rendering with
**one sub-bullet per dependency, each containing only a transcluded block link** to the dependency task, e.g.:

```markdown
- [ ] #task Dispute jacket delivery! [dependsOn:: unemployment] ^call-north-face
  - ![[#^unemployment]]
```

Four deliverables:

1. **`<ctrl+shift+p>` keymap** (the `set-bullet-property` command in `bob-navigation-hotkeys`): stop writing the
   `🔗 **DEPENDS ON:**` bullet; write per-dependency transcluded-block-link sub-bullets instead.
2. **Vault migration**: convert every existing task in `~/bob/` that has dependencies to the new style (both converting
   existing `DEPENDS ON` bullets and synthesizing missing sub-bullets from bare `[dependsOn::]` properties).
3. **`!` keymap** (transclusion toggle in `bob-navigation-hotkeys`): when toggled on a sub-bullet that contains only a
   block link to another Obsidian task, additionally add/remove the corresponding id in the parent task's
   `[dependsOn::]` property; when transcluding (adding `!`), also set `[id:: <block-id>]` on the linked-to task if
   missing.
4. **`bob mark-next-tasks`** (Rust, bob-cli): recursively follow transcluded block links from Pomodoro-linked tasks to
   their dependency tasks and mark those `[*]` too.

The `[dependsOn::]` inline property **remains the machine-readable source of truth** (Obsidian Tasks blocked/blocking
semantics, `block-id-prompt`'s "Blocked" picker chips, and `task-status-cycler`'s generated-id normalizer all read it).
The per-dependency bullets are the human-facing navigation/rendering layer, kept in sync with the property by the plugin
flows.

## Repositories involved

- **bob-plugins** (linked repo; implementers must access it via
  `sase workspace open -p bob-plugins -r "<reason>" <your-workspace-number>` and use the printed path):
  - `plugins/bob-navigation-hotkeys/main.js` — owns both keymaps and all `DEPENDS ON` bullet machinery.
  - `plugins/block-id-prompt/main.js`, `plugins/task-status-cycler/main.js` — read/rewrite `[dependsOn::]` (property
    only; **no changes needed**, but regression-test that they still work).
  - `scripts/` — Node test harnesses (`npm test` runs
    `node --test scripts/test-navigation-hotkeys.cjs scripts/test-task-status-cycler.cjs`); the migration script will
    live here.
- **bob-cli** (primary repo): `src/native/mark_next.rs`, `docs/mark-next-tasks.md`, `tests/cli.rs`,
  `tests/fixtures/mark_next/`.
- **`~/bob/` vault** (data migration target; it is a git repo — do NOT commit to it without Bryan's approval; leave
  changes for review/Obsidian Sync).

## Current state (verified findings)

### bob-navigation-hotkeys (`plugins/bob-navigation-hotkeys/main.js`)

- **`!` toggle**: physical-keydown handler `handleCountedTransclusionTogglePhysicalKeydown` (~line 9214) fires only in
  Vim normal mode **with an explicit count prefix** (`N!`, e.g. `1!`); a count-less palette command
  `toggle-line-transclusions` (~8267) toggles the current line. Both route through `findTransclusionToggleTargets`
  (~4640) / `parseTransclusionToggleTargetAt` (~4665) and `toggleLineTransclusions`/`toggleLineRangeTransclusions`
  (~4865/~4908): if **all** link targets on a line are transcluded, the leading `!` is removed from each; otherwise `!`
  is inserted before each non-transcluded link.
- **`<ctrl+shift+p>`**: the binding itself lives in the vault's `.obsidian/hotkeys.json` (not in the repo) and maps to
  command id `set-bullet-property` (~8273). For the property configured with `values: "local_task_id"` (i.e.
  `dependsOn`), the flow is: pick open local tasks (multi-select) → write `[dependsOn:: id, id]` on the current task
  (`upsertLocalTaskIdValue` ~1543, `applyLocalTaskDependencyListEdits` ~1683) → ensure each target has a trailing
  `^block-id` and an `[id::]` property (`resolveTargetTaskIdentity` ~790, `applyPromptedBlockIdToTaskLine` ~785,
  prompting for a block id when absent) → insert/update the managed navigation bullet
  (`planDependencyNavigationBulletSync` ~1191, `applyDependencyNavigationBulletSyncPlan` ~1331, driven from
  `setLocalTaskDependency` ~8853).
- **Managed bullet format** (constants ~188-210, `formatDependencyNavigationBulletWithMarker` ~858):
  `<indent>- 🔗 **DEPENDS ON:** [[#^a]] • [[#^b]]` — recognized by `DEPENDENCY_NAVIGATION_BULLET_RE` (~202) with a
  legacy label set (`DEPENDENCIES`). Related plumbing: `collectDependencyNavigationBullets` (~1058),
  `reconcileDependencyNavigationBullets` (~7989), and the `consolidate-dependency-navigation-links` command
  (~8279/~8533).
- Helpers are exported for tests via `module.exports.helpers` (~12281); tests stub the `obsidian` module.

### Vault (`~/bob/`)

- 41 real `[dependsOn::]` property instances across 18 files; only 20 `DEPENDS ON` sub-bullets across 8 files — **~21
  dependsOn tasks have no sub-bullet at all** (including `done/obsidian_done.md`, which uses old raw 6-char block-id
  values with no `[id::]` on targets, and `sase_config.md`, which has `dependsOn` on plain non-checkbox bullets). No
  `DEPENDS ON` bullet exists without a matching property.
- Conventions: target tasks carry `[id:: X]` + trailing `^X` (same slug); dependency links are same-file `[[#^X]]`;
  multi-dep bullets join with `•`; child indentation is inconsistent (2-space and TAB both occur, even within one file).
  Value spacing varies (`[dependsOn:: x]` vs `[dependsOn::x]`).
- Transcluded block links (`![[note#^id]]`, 243 occurrences) are already the established convention for Pomodoro ledger
  references and `#^ref` reference notes — **the reconcile logic must never treat those as dependency bullets** (e.g.
  `- ![[ref/chat/foo#^ref]]` nested under a task must not be deleted as "stale").

### bob-cli `mark-next-tasks` (`src/native/mark_next.rs`, ~1770 lines)

- Self-contained: bespoke task parser (`parse_task_line` ~1086; status char, trailing `^block-id`, global filter `#task`
  from the Tasks plugin config), bespoke link parser (`block_link_occurrences` ~649, which already detects the `!` embed
  marker), local `NoteIndex` (~1176) resolving exact path → unique case-insensitive basename.
- Algorithm (`sync_next_tasks` ~300): read today's daily note (`BOB_DAY_FILE` / `<vault>/YYYY/YYYYMMDD.md`), find
  `## Pomodoros`, collect block links from child bullets of **open** top-level entries, scan the vault (skipping
  dot-dirs, `done/`, `_generated/`, `_templates/`), then apply the transition matrix: `[ ]`+referenced→`[*]`,
  `[*]`+not-referenced→`[ ]`, `[*]`/`[/]`+referenced kept, done/other unchanged. Separately embeds (`!`-prefixes) and
  relocates completed-task bullets within the daily note. Guard rails: missing daily note / missing Pomodoros section /
  multiple open timed Pomodoros → exit 1 before any write. Atomic writes; `--dry-run`; human/JSON output.
- **No `dependsOn` or dependency-graph logic exists here at all.** (The only dependency traversal in the repo is the
  unrelated dataview/tasks query engine under `src/native/dataview/tasks/`.)

## Design

### D1. New dependency sub-bullet format

```markdown
<indent>- ![[#^<block-id>]] (same-file target) <indent>- ![[<basename>#^<block-id>]] (cross-file target)
```

- Exactly one dependency per sub-bullet; the bullet's entire content is the single transcluded block link (no label, no
  emoji). Bullets sit directly beneath the parent task line (same slot the old `DEPENDS ON` bullet used), one per
  dependsOn id, in the property's order.
- Indentation: reuse the parent's existing child indent when children exist (same rule the plugin already uses via its
  child-indent helper); otherwise parent indent + one TAB (Obsidian default).
- **Managed-bullet recognition rule (safety-critical):** a sub-bullet is a _managed dependency bullet_ iff (a) its
  content is exactly one transcluded block link and nothing else, AND (b) its link's block id (or the target task's
  `[id::]` value) is a member of the parent task's current-or-being-edited `dependsOn` id set. Rule (b) is what protects
  `#^ref`-style reference transclusions from being rewritten or deleted by sync/reconcile.
- Old `🔗 **DEPENDS ON:**` / legacy `DEPENDENCIES` bullets stay _recognized_ (keep the existing regex + legacy label
  set) but are now themselves legacy: any sync/reconcile/consolidate touching a task replaces them with per-dependency
  bullets.

### D2. bob-navigation-hotkeys: `<ctrl+shift+p>` / dependency-bullet machinery rework

In `plugins/bob-navigation-hotkeys/main.js`:

- Replace the bullet _rendering_ (`formatDependencyNavigationBullet*`) with a per-id formatter producing the D1 format
  (same-file ids → `![[#^id]]`; support a cross-file form when the target lives in another note).
- Rework the _plan/apply_ layer (`collectDependencyNavigationBullets`, `planDependencyNavigationBulletSync`,
  insertion/removal/normalization planners, `applyDependencyNavigationBulletSyncPlan`) to manage a **set of bullets**
  instead of a single line: compute desired id list → insert missing bullets, delete bullets for removed ids, convert
  any legacy `DEPENDS ON`/`DEPENDENCIES` bullet encountered, leave unmanaged siblings untouched, keep the operation
  idempotent.
- `setLocalTaskDependency` (~8853) and `reconcileDependencyNavigationBullets` (~7989) keep their responsibilities
  (property write + target `^block-id`/`[id::]` assignment are unchanged); only the bullet-sync step changes.
- Repurpose the `consolidate-dependency-navigation-links` command to rewrite a whole note to the new canonical format
  (useful as an in-editor escape hatch and as the migration's per-note core).
- Update the plugin's header-comment format documentation and bump `manifest.json` (1.9.0 → 1.10.0).

### D3. bob-navigation-hotkeys: `!` keymap dependency sync

Hook the dependency side-effects into the transclusion toggle (both the counted `N!` physical-key path and the
`toggle-line-transclusions` palette command; for counted ranges, evaluate each line independently):

- A toggled line qualifies for dependency sync iff: it is a sub-bullet whose content is exactly one block link
  (`[[...#^id]]` / `![[...#^id]]`), its nearest less-indented ancestor list item is a Markdown checkbox task line, and
  the link resolves to a task line carrying that `^block-id` (same file: scan the buffer; cross file: resolve via
  `app.metadataCache`/vault, mirroring how the plugin resolves link targets elsewhere). Non-qualifying lines keep
  today's plain-toggle behavior with no side effects.
- **On transclude (adding `!`):** compute the dependency id — the target task's `[id::]` value when present, else the
  block id — and add it to the parent task's `[dependsOn::]` (create the property if missing; append with `", "`;
  dedupe; reuse `upsertLocalTaskIdValue`/`applyLocalTaskDependencyListEdits`). If the target task lacks an `[id::]`
  property, set `[id:: <block-id>]` on it (the block id already exists by construction). Cross-file target edits go
  through `app.vault.process`.
- **On un-transclude (removing `!`):** remove the corresponding id from the parent's `[dependsOn::]` list (matching
  either the target's `[id::]` value or the block id); when the list empties, remove the whole `[dependsOn::]` field.
  Leave the target's `[id::]` and block id in place.
- Editor edits must be applied as one transaction per file so undo behaves sanely; guard against double-applying when
  the same target id is already present/absent.

### D4. Vault migration (one-off, idempotent)

New script `scripts/migrate-dependency-bullets.mjs` in bob-plugins (dry-run by default, `--write` to apply,
`--vault <dir>` defaulting to `~/bob`). Prefer driving it through the plugin's exported `module.exports.helpers` (as the
existing `.cjs` tests do, stubbing `obsidian`) so migration and runtime share one format implementation; fall back to a
standalone implementation only if the helper surface proves impractical.

Behavior, per Markdown file (skip `.git`, `.obsidian`, `_generated`, `_templates`; **include** `done/`):

1. Find every list-item line carrying `[dependsOn:: ...]` (checkbox tasks _and_ plain bullets — e.g. `sase_config.md`
   has plain-bullet dependencies).
2. Resolve each id vault-wide: prefer a task line with `[id:: <id>]`, else a line with trailing `^<id>`; the link
   fragment always uses the target's actual trailing block id. Same file → `![[#^id]]`; other file →
   `![[<basename>#^id]]`.
3. Remove any existing managed `🔗 **DEPENDS ON:**` / legacy bullets under the item; insert per-dependency bullets per
   D1 (reuse the removed bullet's indent when converting; else existing-child indent; else TAB).
4. Unresolvable ids (likely some `done/` targets): leave the property untouched, create no bullet, and report the
   file/line/id in the summary — never guess.
5. Print a per-file change summary + totals; idempotent (second run = no-op).

Execution: run dry-run, sanity-check output against the known counts (41 properties / 20 old bullets / 18 files), run
with `--write`, then verify with `git -C ~/bob diff` and spot-check in Obsidian. Leave the vault changes uncommitted for
Bryan's review.

### D5. bob-cli: recursive dependency promotion in `mark-next-tasks`

In `src/native/mark_next.rs`:

- **Parse dependency edges:** extend the vault scan so each parsed task also records candidate dependency references
  found in its child block (lines below it with greater indentation, up to the next line at or below the task's indent,
  skipping fenced code): sub-bullets whose content is exactly one **transcluded** block link (`![[...#^id]]`) — matching
  the D1 managed format. Plain `[[#^id]]` links are deliberately _not_ edges (the `!` toggle is what activates a
  dependency).
- **Resolve + filter:** resolve candidates through the existing `NoteIndex`; keep only edges whose target is a parsed
  task line with that block id (this drops `#^ref` reference transclusions naturally). Unresolvable dependency links get
  an `unresolved_references`-style warning that names the referencing task's file (note: `done/` is outside the scan, so
  edges into `done/` will surface as warnings — acceptable, since completed dependencies are terminal anyway).
- **Closure:** BFS from the directly-Pomodoro-referenced task set over dependency edges, with a visited set
  (cycle-safe). The resulting closure replaces the `desired`/`referenced` set feeding the existing transition matrix —
  so dependencies of a linked task are promoted `[ ]`→`[*]`, kept if already `[*]`/`[/]`, left alone if done/cancelled,
  and (critically) **not** cleared by the stale-`[*]` reset; conversely, when a task loses its Pomodoro link, its whole
  dependency chain clears unless reachable from another linked task.
- **Output:** count dependency-derived references separately (e.g. a `dependency_references` JSON field alongside
  `references`) and distinguish dependency-promoted tasks in human output (e.g. a `(dependency)` suffix). Keep the rest
  of the JSON shape stable.
- **Docs/help:** update `docs/mark-next-tasks.md` (reference source, transition, warnings, JSON schema, examples) and
  the clap `about`/long-help in `build_cli` (~58-110). No new CLI options are needed; per CLI rules, keep help text
  excellent and option lists alphabetical.

## Testing & verification

- **bob-plugins** (`npm test`, plus `npm run validate` for manifests): add node tests covering — the new bullet
  formatter/recognizer (incl. cross-file form and the "sole transcluded link + id ∈ dependsOn" managed rule); sync-plan
  insert/rewrite/delete/convert-legacy/no-op cases; protection of `#^ref` transclusions and arbitrary child bullets;
  `!`-toggle dependency sync (add id / create property / remove id / drop empty property / set missing `[id::]` on
  target / non-qualifying lines untouched); counted-range toggling across mixed lines. Current coverage of these helpers
  is thin, so add characterization tests for existing behavior being preserved (e.g. plain multi-link line toggling).
- **Migration:** unit-test the transform against fixture texts (old bullet → new bullets; property-only task gains
  bullets; multi-dep `•` split; TAB vs 2-space indents; plain-bullet dependsOn; unresolvable id skipped; idempotency).
  Then the dry-run/diff verification on the real vault per D4.
- **bob-cli** (`just` = fmt + lint + test): extend `tests/fixtures/mark_next/` with dependency chains and add
  integration tests — same-file dep promoted; cross-file dep promoted; dep-of-dep (recursion); cycle terminates; plain
  (non-transcluded) sub-bullet link NOT promoted; transcluded non-task `#^ref` link ignored; completed dependency left
  `[x]`; stale chain cleared when the Pomodoro link disappears; JSON field assertions; idempotency of the main fixture
  test preserved.
- **End-to-end:** after `bob plugins sync -r <bob-plugins-workspace-root>` deploys the plugins, exercise the real flows
  via a Pomodoro-linked task with a dependency chain and run `bob mark-next-tasks --dry-run --format json` against the
  migrated vault to confirm sensible promotions before a real run.

## Sequencing

1. bob-navigation-hotkeys rework (D1-D3) + plugin tests → deploy via `bob plugins sync`.
2. Migration script (D4) + tests → dry-run → `--write` on `~/bob/` → diff review.
3. mark-next-tasks recursion (D5) + docs + tests (fixtures use the new, migrated format).
4. Final end-to-end check on the live vault (dry-run first).

## Risks / edge cases to honor

- **Never delete non-dependency transclusions**: the "id ∈ dependsOn set" recognition rule is the invariant; test it
  explicitly.
- Old `done/` dependencies use raw 6-char ids without `[id::]` — resolve by block id; report (don't touch) the
  unresolvable ones.
- Mixed TAB/2-space child indents in the vault — always mirror local context rather than normalizing globally.
- `[dependsOn::x]` (no space) must parse everywhere the property is read (the existing regexes already tolerate this;
  keep it that way).
- Multiple links on one line, or extra text around a link, disqualify a bullet from dependency semantics — plain toggle
  only.
- The vault is the user's live data: migration must be dry-run-first, idempotent, and report everything it skips; no
  vault git commits without approval.
