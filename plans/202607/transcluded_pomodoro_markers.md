---
create_time: 2026-07-12 10:43:52
status: done
prompt: 202607/prompts/transcluded_pomodoro_markers.md
tier: tale
---

# Plan: Keep Pomodoro Markers off Transcluded Task History

## Goal

Correct the Pomodoro marker semantics so the tomato records a non-transcluded task link that participated in a completed
Pomodoro, while transcluded task links retain the existing completion/retirement behavior without receiving a tomato.
Closing a Pomodoro with `<Ctrl+Enter>` should produce these distinct histories:

- A non-transcluded source link is marked under the completed Pomodoro and copied forward without the marker:
  `[[dev#^continue]]` becomes `🍅 [[dev#^continue]]`, with `[[dev#^continue]]` in the next Pomodoro.
- A transcluded source link has its target task tree completed recursively, then the source token is un-transcluded and
  struck without a marker: `![[dev#^finish]]` becomes `~~[[dev#^finish]]~~`.
- If a previously marked non-transcluded link is retired later, its marker is preserved: `🍅 [[dev#^continue]]` becomes
  `🍅 ~~[[dev#^continue]]~~`.

The plugin and `bob mark-next-tasks` must agree on this behavior so a later CLI synchronization does not re-add a tomato
that the keymap correctly omitted.

## Diagnosis and root cause

The screenshot and the live `2026/20260712` daily note show the incorrect output on the two links that were transcluded
before the 0900–0950 Pomodoro was closed:

- `🍅 ~~[[bob#^wip-pom-icon]]~~`
- `🍅 ~~[[sase#^fix-sdd-error]]~~`

This is deterministic behavior introduced by the original Pomodoro-marker feature, not an editor timing or rendering
problem:

1. In `task-status-cycler`, `getBlockLinkTokenCandidates` deliberately combines embedded and non-embedded candidates.
   `buildPomodoroCompletionPlan` then calls `rewritePomodoroMarkersInLine(..., true)` across every source sub-bullet, so
   it marks `![[...]]` before the retirement pass has a chance to un-transclude and strike it.
2. `retireClosedTaskReferencesInText` reinforces the mistake. After retiring matching embeds beneath a done Pomodoro, it
   calls the same all-links marker rewrite for the whole line, so even a retirement path that began unmarked becomes
   marked.
3. The focused plugin tests explicitly expect both `🍅 ![[...]]` and the resulting `🍅 ~~[[...]]~~`, confirming that the
   broad behavior is encoded in the current specification tests.
4. `bob mark-next-tasks` independently uses `entry.completed` as the sole `marker_expected` decision for every link
   occurrence. It therefore adds markers to embedded anomalies and all struck links beneath completed Pomodoros. A
   plugin-only fix would be unstable because the next CLI run could restore the unwanted marker.
5. Once an embedded link has been retired, Markdown only retains `~~[[...]]~~`; that token does not reveal whether it
   originated as an embed or as a non-transcluded link that was completed later. Historical repair must therefore avoid
   guessing provenance from a struck wikilink.

The implementation should replace the old ownership rule (“marked iff beneath a done Pomodoro”) with a per-occurrence,
state-aware rule that uses transclusion state while it is still observable and preserves provenance afterward.

## Corrected marker contract

### Completed Pomodoros

- Live, non-transcluded block links are canonically prefixed with one `🍅 `.
- Embedded block links are canonically unmarked. If their resolved task is complete, retirement removes `!`, adds the
  strike envelope, and still leaves them unmarked.
- A struck, non-embedded link with an existing marker keeps one canonical marker. This preserves the history of a link
  that began non-transcluded and was completed later.
- A struck, non-embedded link without a marker stays unmarked. The missing marker may be the only surviving evidence
  that it was retired from a transclusion, so neither writer may synthesize one.
- Mixed-content bullets apply these decisions independently to each link token. Aliases, prose, indentation, nested
  descendants, and line endings remain unchanged.

### Open and cancelled Pomodoros

- Open Pomodoros keep the existing clean-copy behavior: task links are unmarked, including copies inserted by
  `<Ctrl+Enter>`.
- Cancelled Pomodoros remain outside marker normalization, retirement, promotion, and relocation changes.

### Reporting and historical data

- Marker add/remove reporting continues to describe only real token changes. Removing a stray marker from a still-
  embedded token is a marker removal; omitting a marker while retiring an unmarked embed is not.
- Do not perform a global migration over existing struck links because their origin is no longer inferable. After both
  writers implement the provenance-preserving rule, remove the tomatoes from the two user-confirmed affected links in
  `2026/20260712`; a follow-up CLI dry run must leave them unmarked.

## Implementation approach

### `bob-plugins`: Task Status Cycler

1. Refine the pure marker rewrite helpers in `plugins/task-status-cycler/main.js` so callers can choose marker state per
   parsed occurrence rather than supplying one boolean for every block link on a line. Continue using the existing
   embedded/non-embedded parsers, exact strike spans, right-to-left edits, duplicate normalization, fence handling, and
   EOL-preserving text helpers.
2. In `buildPomodoroCompletionPlan`, normalize each source sub-bullet from its pre-completion syntax:
   - add/canonicalize the marker for non-embedded live candidates;
   - remove/omit it for embedded candidates;
   - preserve the marker state of already-struck candidates instead of inventing missing provenance. Build copy-forward
     lines from marker-stripped source text as today, retaining the existing eligibility, dedupe, placeholder, and
     cursor behavior.
3. Make `retireClosedTaskReferencesInText` candidate-aware. A matching embedded reference under a done Pomodoro should
   be rewritten directly to the unmarked struck form. Remove the final whole-line “mark everything” pass. Retirement of
   a link that already has a valid marker must preserve it, while `#task` ancestry and open-Pomodoro retirement must
   keep their current no-new-marker behavior.
4. Keep recursive task closure sequencing unchanged: resolve and close transcluded target trees first, build/apply the
   local completion plan from the refreshed editor text, then retire closed references. The change is limited to local
   marker decisions around those already-established operations.
5. Update the plugin README description and bump the Task Status Cycler patch version. Deploy only this plugin after the
   complete test suite passes, using a scoped sync preview before applying the sync.

### `bob-cli`: `mark-next-tasks`

6. Replace the single `marker_expected = final_entry.completed` boolean in `plan_structural_changes` with a helper that
   derives the desired marker state from the final Pomodoro state and the occurrence's pre-edit syntax:
   - completed + embedded: unmarked;
   - completed + live non-embedded: marked;
   - completed + already-struck non-embedded: preserve whether a marker exists, canonicalizing an existing marker but
     not adding a missing one;
   - open: unmarked;
   - cancelled: untouched.
7. Apply the same policy when retirement and movement happen together. A completed embedded link moved into the last
   completed fallback becomes unmarked `~~[[...]]~~`; a completed non-transcluded live link moved there becomes marked
   `🍅 ~~[[...]]~~`. Moves into the current open timed Pomodoro remain unmarked. Preserve the current mixed-live-link
   guard and all bullet/subtree relocation behavior.
8. Keep the occurrence model's marked/unmarked and retired token variants, but ensure the chosen replacement and
   `marker_added_references` / `marker_removed_references` accounting use the new per-occurrence policy. The no-op and
   dry-run decisions must remain idempotent when an unmarked struck link exists beneath a completed Pomodoro.
9. Revise `docs/mark-next-tasks.md` and the README to replace the old “every block link under a completed Pomodoro”
   invariant with the corrected grammar and explain why unmarked struck links are preserved rather than backfilled.

## Tests

### Plugin tests

Update `scripts/test-task-status-cycler.cjs` with regressions for:

- pure per-occurrence normalization of plain, embedded, struck, aliased, duplicate-marked, and mixed links;
- Pomodoro completion producing `🍅 [[plain]]` for non-transcluded originals, `~~[[embed]]~~` for transcluded originals,
  and marker-free copies for carried-forward links;
- an erroneously marked embed being normalized to unmarked before/while it is retired;
- retirement beneath a done Pomodoro not adding a marker to an embed, while preserving a marker already attached to a
  non-transcluded reference that closes later;
- active-editor and vault-backed retirement paths, recursive transclusion closure, fences, CRLF, and a second-pass
  no-op;
- the full `<Ctrl+Enter>` flow with both link kinds on separate and mixed-content bullets, including unchanged
  placeholder/cursor behavior and no copy of the retired embed.

Keep the navigation regression proving tomato-prefixed task history is not treated as a managed dependency bullet. Run
`npm test` and `npm run validate` in `bob-plugins`.

### CLI tests

Update unit tests in `src/native/mark_next.rs` and CLI fixtures/assertions in `tests/cli.rs` to cover:

- completed-Pomodoro normalization for live plain, live embedded, marked struck, and unmarked struck occurrences;
- retirement of completed embedded references without a marker and completed non-embedded references with the correct
  marker when moved to a completed fallback;
- open-Pomodoro marker removal, cancelled-Pomodoro non-interference, mixed bullets, aliases, CRLF, and nested moves;
- stable JSON/human marker reporting and a second-run no-op where unmarked struck history is canonical;
- interoperability fixture output matching the plugin's `<Ctrl+Enter>` result so `mark-next-tasks` does not re-add the
  tomato;
- continued preservation of valid markers during `move-done-tasks` archive retargeting.

Run the repository-wide `just all` gate.

## Live verification and cleanup

1. In a scratch daily note, create one plain block link and one transcluded block link under an open Pomodoro, then use
   `<Ctrl+Enter>`. Confirm the target behind the embed is completed, the source becomes unmarked `~~[[...]]~~`, the
   plain source becomes marked, and only the plain source is copied forward without its marker.
2. Run `bob mark-next-tasks --dry-run --format json` against the scratch/live vault and confirm it proposes no marker
   addition for the unmarked struck history.
3. Remove the tomatoes from the two user-confirmed affected links in `2026/20260712` only. Do not infer or rewrite other
   historical struck links.
4. Run a final CLI dry run and confirm those links remain unchanged and the overall operation is idempotent.

## Expected files

### `bob-plugins`

- `plugins/task-status-cycler/main.js`
- `plugins/task-status-cycler/manifest.json`
- `scripts/test-task-status-cycler.cjs`
- `README.md`

### `bob-cli`

- `src/native/mark_next.rs`
- `tests/cli.rs`
- `tests/fixtures/mark_next/2026/20260710.md`
- `docs/mark-next-tasks.md`
- `README.md`

The live daily note `2026/20260712` receives only the targeted cleanup after both code paths are fixed and verified.

## Sequencing and risk controls

1. Implement and test the plugin's per-occurrence policy first, preserving the existing completion/retirement ordering.
2. Mirror the same policy in the CLI planner and reporting, then run the full CLI gate.
3. Run both tools against shared scratch cases to prove interoperability and idempotence.
4. Deploy the plugin, perform the narrowly scoped live cleanup, and verify with a final dry run.

The principal risk is loss of transclusion provenance after `![[...]]` becomes `~~[[...]]~~`. The plan avoids unsafe
inference by making the decision before retirement and treating an unmarked struck link as a valid stable state. Other
risks—marking a copied line, disrupting recursive closure, changing mixed-bullet moves, or reintroducing markers through
the CLI—are isolated with exact-output and second-pass tests in both repositories.
