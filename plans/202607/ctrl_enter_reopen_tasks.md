---
create_time: 2026-07-12 11:46:25
status: done
prompt: 202607/prompts/ctrl_enter_reopen_tasks.md
tier: tale
---

# Plan: Make Ctrl+Enter Reopen Closed Obsidian Work Reliably

## Goal

Make the Task Status Cycler plugin's normal-mode `<Ctrl+Enter>` action a reliable open/done toggle for every supported
selection shape: an ordinary task checkbox, a Pomodoro checkbox, or a block link whose resolved source is a task. When
the selected work is Done (`[x]`/`[X]`), reopen it as Todo (`[ ]`). Reopening a Pomodoro should also reopen each
directly linked Done source task beneath that Pomodoro. Reopening through a task link must stop at the resolved root; it
must not walk that task's child transclusions.

Preserve the established forward behavior: Todo, Next, and In Progress tasks close to Done; completing an open Pomodoro
still closes embedded task trees recursively, starts strict bare task links, creates or selects the next Pomodoro,
carries eligible links forward, records markers, and retires newly closed references.

## Findings and constraints

- Both `<C-CR>` and `<C-Enter>` already map to `taskStatusCyclerToggleTaskOpenDone`; registration is not the failure.
- A directly selected Done task already toggles to Todo. The gap is dispatch: a Done Pomodoro is handled by the generic
  local checkbox path, so none of its linked source tasks reopen.
- An embedded link beneath a Pomodoro is always routed into recursive forced completion. A Done resolved root is
  intentionally reported as handled to prevent the generic toggle from reopening it. This makes `<Ctrl+Enter>` a no-op
  for exactly the reported closed-target case.
- Task closure retires managed embeds to non-embedded struck links such as `~~[[Tasks#^work|Work]]~~`. Supporting
  completed Pomodoro history therefore requires the selection/collection model to recognize embedded, plain, and
  canonical retired block links; looking only for `![[...]]` cannot make reopening reliable.
- Source-task writes already have the important safety properties to retain: path-plus-block-ID resolution, metadata
  cache fallback scanning, standalone block-ID revalidation, live-editor writes for the active note, guarded vault
  processing for other notes, and removal of stale completion metadata when a task leaves Done.
- Reopening is a status operation, not an inverse reference-retirement migration. Preserve Pomodoro markers,
  strikethroughs, aliases, and embed/plain spelling in the referencing note, and do not globally resurrect retired
  dependency links. The action only changes the selected Pomodoro/task and the directly corresponding source task
  statuses.
- The linked plugin repository currently has a green baseline: all 76 tests pass and all six plugin manifests validate.
  Its SDD companion contains an unrelated user-owned status edit, which must remain untouched.

## Behavioral contract

### Direct task selection

- Done becomes Todo through the existing Tasks-aware setter, so `#task` metadata is normalized by the Tasks command or
  the metadata-aware fallback and ordinary checkboxes keep their local rewrite behavior.
- Todo, Next, and In Progress continue to become Done and trigger the existing reference-retirement workflow.
- Canceled and unsupported custom statuses remain ineligible.

### Direct Pomodoro selection

- An open Pomodoro keeps the current completion workflow unchanged.
- A Done top-level checkbox inside `## Pomodoros` becomes Todo. Before or after that local write, resolve every valid
  block-link occurrence in its immediate sub-bullet block and reopen each directly resolved Done task as Todo.
- Recognize embedded (`![[...#^id]]`), plain (`[[...#^id]]`), Pomodoro-marked, and retired struck (`~~[[...#^id]]~~`)
  occurrences, including aliases, same-note links, and cross-note links. Dedupe repeated occurrences by resolved
  Markdown path plus block ID.
- Only Done linked roots change. Already-open, Next, In Progress, canceled, custom, malformed, ambiguous, and unresolved
  targets remain untouched. One failed target must not prevent independent siblings or the Pomodoro checkbox from
  reopening.
- Do not follow embedded block links found inside any resolved source task's descendant list block. No placeholder is
  inserted, no links are carried forward, no cursor jump/centering runs, and no close-reference retirement runs on a
  reopen.

### Block-link selection

- Resolve the cursor-selected task block link once, using the current cursor-aware ambiguity rule: choose the token
  under the cursor, or accept the line only when it has one unambiguous eligible candidate.
- If the resolved root is Done, reopen that root as Todo immediately and report the action handled. This branch has
  priority over Pomodoro-child recursive completion and never examines descendants.
- If the resolved root is incomplete, retain the existing context-sensitive paths: an embedded Pomodoro child closes
  recursively, a strict bare link beneath an open Pomodoro starts only its root, and a generic task link toggles only
  its root.
- Allow canonical retired struck links to select their resolved task without changing the link's historical rendering.

## Implementation approach

1. **Separate reopen policy from close/traversal policy in `task-status-cycler`.** Add explicit predicates/results for a
   Done target that can reopen, rather than broadening the existing forced-status rules used by `/ -> [ ]` and recursive
   completion. Introduce a root-only resolved-target reopen helper that forces `[x] -> [ ]` through the existing
   revalidated editor/vault write machinery and reports whether the target was resolved and changed.

2. **Generalize task block-link discovery for reopen decisions.** Reuse the existing block-link token parsers and
   strikethrough span model to expose selectable embedded, plain, marked, and retired candidates while preserving the
   current path/block validation and cursor disambiguation. Keep the stricter existing recognizers for open-Pomodoro
   start and recursive-close semantics so broad parsing does not accidentally turn prose or mixed bullets into new
   close/start actions.

3. **Make the Ctrl+Enter dispatcher status-aware after resolution.** In the Vim action, detect a Done Pomodoro before
   the generic checkbox toggle. For link selections, resolve the selected root and branch on its actual status: reopen
   Done roots non-recursively; otherwise delegate to the current recursive Pomodoro-child, bare-link start, or generic
   toggle flow. Avoid double resolution where practical, and preserve the existing best-effort error behavior and
   handled/fallthrough semantics.

4. **Add a root-only completed-Pomodoro reopen workflow.** Locate the selected Done Pomodoro's sub-bullet range, collect
   all direct block-link candidates from those lines, resolve and dedupe them, and reopen only Done roots. Apply the
   Pomodoro's own Done-to-Todo transition through the existing local/Tasks-aware path without invoking completion-plan,
   carry-forward, centering, marker-rewrite, or retirement code. Keep cross-file writes serialized or sequential so
   active-editor content and vault-backed sources use their correct write paths.

5. **Document and version the user-facing behavior.** Update the Task Status Cycler comments and repository plugin
   summary to describe symmetric Ctrl+Enter close/reopen semantics and the deliberately asymmetric recursion rule
   (recursive close, root-only reopen). Bump the Task Status Cycler patch version in its manifest and the README table.

## Tests and verification

Extend `scripts/test-task-status-cycler.cjs` with focused pure and runtime regressions:

- Status policy: Done is reopenable to Todo; Todo, Next, In Progress, canceled, and custom statuses are not mistaken for
  reopen candidates, while the existing direct close matrix remains unchanged.
- Selection parsing: embedded, plain, aliased, Pomodoro-marked, and retired struck block links resolve correctly; cursor
  selection disambiguates mixed/multiple-link lines; malformed, heading-only, fenced, and ambiguous candidates are
  rejected where applicable.
- Direct task: both registered Ctrl+Enter key spellings still dispatch the same action; a Done `#task` uses the Tasks
  Todo command/fallback and does not trigger retirement.
- Selected transclusion: a Done embedded Pomodoro child reopens only its root while a nested Done descendant remains
  Done; cover both same-file active-editor and cross-file vault-backed targets. Preserve an explicit regression proving
  an incomplete embedded Pomodoro child still closes recursively.
- Retired selection: selecting `~~[[...#^id|alias]]~~` reopens the resolved Done root without altering the referencing
  line's strike/embed/marker text and without following descendants.
- Done Pomodoro: the checkbox and every directly linked Done root reopen; duplicate links cause one source write;
  already-open/in-progress/next/canceled/custom and unresolved roots remain unchanged; linked descendants remain Done;
  link text, marker/strike history, cursor position, existing later Pomodoros, and carried-forward bullets are
  unchanged.
- Failure isolation: one unreadable or stale linked source does not prevent the Done Pomodoro or other resolvable roots
  from reopening.

Run the linked repository's complete checks:

```bash
npm test
npm run validate
```

Then deploy the source-of-truth plugin through the required workflow, using the opened linked repository as the source
and avoiding a pull over local implementation changes:

```bash
bob plugins sync --no-pull -p task-status-cycler -r <linked-bob-plugins-repository>
```

Finally, manually exercise a scratch note in Obsidian: reopen a Done ordinary task, a Done retired/transcluded target
whose source has a Done child, and a Done Pomodoro containing several direct task links. Confirm only the
selected/direct roots become Todo, descendants remain Done, historical link formatting is preserved, and pressing
Ctrl+Enter on the same shapes while incomplete still follows the established close/start behavior.

## Expected files

- `plugins/task-status-cycler/main.js`
- `plugins/task-status-cycler/manifest.json`
- `scripts/test-task-status-cycler.cjs`
- `README.md`

No `bob-cli`, vault-note, navigation-plugin, or SDD memory changes are required for this behavior.
