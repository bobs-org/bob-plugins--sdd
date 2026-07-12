---
create_time: 2026-07-12 08:32:44
status: done
prompt: .sase/sdd/plans/202607/prompts/pomodoro_session_marker.md
tier: tale
---

# Plan: 🍅 Pomodoro Marker for Task Links Under Done Pomodoros

## Goal

Give every task block link that lives in a sub-bullet beneath a **done** Pomodoro in a daily note a beautiful,
machine-owned visual marker: the 🍅 (tomato — the literal "pomodoro") emoji rendered immediately to the left of the
link. The marker tells the story of a session at a glance: "this task rode along in a Pomodoro that finished." Both the
Obsidian `<Ctrl+Enter>` keymap (`task-status-cycler`) and `bob mark-next-tasks` must write, preserve, and repair the
marker while keeping today's retire-as-struck-link contract fully intact:

- When `<Ctrl+Enter>` completes a Pomodoro, the sub-bullets left behind under the newly done Pomodoro get the marker
  added to each of their block links — including the non-transcluded links whose bullets are copied forward to the newly
  created Pomodoro. The **copies never carry the marker**; only the originals do.
- When a linked task is later completed (via `<Ctrl+Enter>` or discovered by `bob mark-next-tasks`), the link is still
  untranscluded and struck exactly as today, and the marker is **kept**: `🍅 [[note#^id]]` → `🍅 ~~[[note#^id]]~~`.
- The marker is an invariant, not a one-shot decoration: links under done Pomodoros are _always_ marked, however the
  Pomodoro came to be done.

## Current behavior (context)

- `task-status-cycler`'s `<Ctrl+Enter>` Pomodoro-completion flow (`completeActivePomodoroTask` →
  `classifyPomodoroSubBullets` → `buildPomodoroCompletionPlan` → `applyPomodoroCompletionPlan` →
  `retireClosedTaskReferences`) recursively completes transcluded (`![[...]]`) sub-bullets, marks the Pomodoro `[x]`,
  inserts a `- [ ] ()` placeholder Pomodoro carrying verbatim copies of every sub-bullet that contains at least one
  non-embedded, non-struck block link, and finally retires references to every task closed by the action as
  `~~[[...]]~~`.
- `task-status-cycler`'s task-closure paths share `retireClosedTaskReferencesInText`, which rewrites qualifying link
  tokens (guarded by `hasEligibleRetirementAncestor`: `#task` ancestry or a top-level checkbox in a `## Pomodoros`
  section) to the canonical struck, non-embedded form.
- `bob mark-next-tasks` (`src/native/mark_next.rs`) scans the daily note's `## Pomodoros` section, promotes/clears Next
  statuses from links under open Pomodoros, retires links resolving to completed tasks as `~~[[...]]~~` beneath both
  open and completed Pomodoros (`block_link_occurrences` + `plan_structural_changes` + `apply_structural_plan`), and
  relocates completed-link bullets from open Pomodoros to the current timed Pomodoro or the last completed fallback.
  Repairs under completed Pomodoros are normalized in place and never moved.
- No icon convention exists on these sub-bullets today. Precedents elsewhere: `bob projects sync` renders `🗓️ [[child]]`
  markers directly before links in its machine-owned ledger line, and Obsidian Tasks reserves 🗓/📅/✅/⏳/🏁 and friends
  as task-line metadata signifiers.

## Product decisions

- **Icon**: `🍅` (U+1F345, single code point, no variation selector). It is the canonical Pomodoro symbol, renders
  crisply in editor and preview, is currently unused across the vault, `bob-cli`, and `bob-plugins`, and collides with
  no Obsidian Tasks emoji signifier. Call it the **Pomodoro marker** in code, docs, and output.
- **Grammar**: the marker is `🍅` followed by a single space, placed immediately before the link token and **outside**
  any strike envelope so the tomato itself is never struck through:
  - carried-forward live link: `- 🍅 [[dev#^write-tests]]`
  - retired link: `- 🍅 ~~[[dev#^write-tests|alias]]~~`
  - anomalous still-embedded link under a done Pomodoro: `- 🍅 ![[dev#^ref]]` (marked positionally; retirement still
    requires proven completion).
- **Placement is per link token, not per bullet.** Mixed-content bullets mark each qualifying link
  (`- Work on 🍅 [[a#^x]] and 🍅 ~~[[b#^y]]~~`), mirroring the per-link `🗓️` sub-projects precedent and composing
  naturally with both codebases' existing token-span edit machinery. Surrounding prose, aliases, indentation, list
  markers, nested descendants, and LF/CRLF are preserved verbatim.
- **Ownership invariant**: in exactly this position — immediately before a block link inside a `## Pomodoros` sub-bullet
  — the marker is machine-owned state meaning "this link belongs to a done (`[x]`/`[X]`) Pomodoro." Both tools normalize
  toward _marked ⟺ under a done Pomodoro_:
  - links under done Pomodoros always gain the marker (self-healing for historical entries and Pomodoros completed by
    hand without the keymap);
  - stray markers under open Pomodoros are removed (covers reopened Pomodoros and accidental copies), and duplicate
    markers collapse to one.
  - Cancelled (`[-]`) Pomodoros are neither open nor done; their bullets stay untouched, exactly like today's retirement
    rules.
- **Scope**: daily-note `## Pomodoros` sub-bullet block links only. Dependency bullets under `#task` lines, the Pomodoro
  entry line itself, prose, fenced code, and every other context never receive the marker.
- **Copies are born clean**: bullet lines copied to the next Pomodoro during `<Ctrl+Enter>` completion are inserted
  without markers (and defensively stripped of any marker they might carry), so a task's history reads as one marked
  entry per finished session plus one clean entry in the live session.

## Behavioral contract

### `<Ctrl+Enter>` Pomodoro completion (`task-status-cycler`)

1. The completion plan gains a marking step for the source Pomodoro's sub-bullets: every valid block-link token (plain,
   embedded, or struck; any alias; any position in the bullet) is prefixed with `🍅 ` unless already marked. This
   happens as part of the same atomic plan that flips the Pomodoro to `[x]` and inserts the new placeholder, so a
   completed Pomodoro is never observable with unmarked links.
2. Copied bullet lines are derived from the **unmarked** source text (markers stripped if somehow present), preserving
   today's copy criteria (non-embedded, non-struck block links) and placeholder/cursor behavior byte for byte.
3. The subsequent reference-retirement pass then strikes the just-closed transcluded targets as usual and must preserve
   the marker it now finds in front of those tokens, producing `🍅 ~~[[...]]~~`.

### `<Ctrl+Enter>` task closure (all retire paths)

4. `retireClosedTaskReferencesInText` preserves an existing `🍅 ` prefix untouched when rewriting a token to the struck
   form (the marker sits outside the edited span; spacing heuristics must not misfire on it).
5. When the retired token's ancestry resolves to a **done** top-level Pomodoro checkbox, the rewrite also adds the
   marker if missing (extend the `hasEligibleRetirementAncestor` gate to report the Pomodoro's checkbox state). Under
   open Pomodoros and `#task` ancestry the output stays exactly as today — no marker.

### `bob mark-next-tasks`

6. Retirement output becomes position-aware: tokens under done Pomodoros canonicalize to `🍅 ~~[[...]]~~`, tokens under
   open Pomodoros to `~~[[...]]~~`. The `canonical` (skip) predicate follows suit: an unmarked struck token under a done
   Pomodoro, or a marked one under an open Pomodoro, is no longer canonical and gets repaired.
7. A marker-repair sweep enforces the invariant across today's daily note: add missing markers to every block-link token
   beneath done Pomodoros (live, struck, or embedded), remove stray markers beneath open Pomodoros, and collapse
   duplicates. Marker repair is pure decoration — it never changes which tasks are promoted, cleared, retired, or moved,
   and links under done Pomodoros still never feed the Next/dependency sets.
8. Relocation composes with marking: a completed-link bullet moved to the **last completed fallback** Pomodoro gains
   markers (it now belongs to a done session); one moved to the current open timed Pomodoro stays unmarked. The existing
   guards (never move out of completed entries, never move mixed live bullets into the fallback) are unchanged.
9. Reporting: human output gets a marker section and summary count alongside struck/moved (e.g. `marked` / `would mark`,
   `unmarked` for removals), and JSON gains stable `marker_added_references` and `marker_removed_references` lists
   (`target`, `block_id`, `pomodoro`). The no-op / `already in sync` decision and dry-run accounting include marker
   edits. Existing fields, warnings, guard rails, atomic writes, and idempotence guarantees are preserved; a second run
   is always a no-op.

### Non-interference guarantees

- The marker only ever exists under done Pomodoros, so every "act on the open Pomodoro" path (bare-link starts,
  transcluded-child completion, Next promotion, copy-forward classification) keeps seeing today's unmarked grammar.
- `bob-navigation-hotkeys` dependency reconciliation manages only bullets that are _exactly_ a lone
  `![[...]]`/`~~[[...]]~~` token; a 🍅-prefixed bullet does not match its anchored recognizer and is left alone. No
  change needed there beyond a regression test.
- `bob move-done-tasks` link repair rewrites only the text inside `[[...]]`, so markers survive archive retargeting.
  Covered by a regression test, no code change expected.
- The native Tasks engine, `bob pomodoro`/`bob tmux-pomodoro` (entry lines only), and `bob capture` never parse these
  tokens and are unaffected.

## Implementation approach

### `bob-plugins` (access via `sase workspace open -p bob-plugins ...`; deploy with `bob plugins sync`)

1. **Pure marker helpers in `plugins/task-status-cycler/main.js`.** Add small, unit-testable helpers: detect a marker
   immediately before a token span (tolerant of extra whitespace), produce marked/unmarked rewrites of a line's link
   tokens (right-to-left span edits, EOL-preserving), and strip markers from a line. Reuse the existing
   embed/wikilink/strike span parsers.
2. **Mark originals during Pomodoro completion.** Extend `buildPomodoroCompletionPlan` (and its application) with
   per-line rewrites that mark every block-link token in the source sub-bullet block, while copied lines are built from
   marker-stripped text. Keep dedupe/no-copy/placeholder semantics untouched.
3. **Marker-aware retirement.** Extend the retirement ancestry gate to surface whether the qualifying Pomodoro ancestor
   is done, thread that through `retireClosedTaskReferencesInText`, and emit the marker on struck rewrites under done
   Pomodoros while preserving any existing marker everywhere.
4. **Docs and version.** Update the plugin's README row/description and bump `plugins/task-status-cycler/manifest.json`.

### `bob-cli`

5. **Marker-aware occurrence model in `src/native/mark_next.rs`.** Teach `block_link_occurrences` to detect a leading
   marker per occurrence (span start, marked flag) and compute canonical tokens for both positions. Thread each bullet's
   owning-entry state (open/completed) into canonicality and token construction.
6. **Plan and apply marker repairs.** Extend `plan_structural_changes` to emit marker-add/remove token edits for
   decoration-only repairs and to mark tokens on bullets relocated into the completed fallback; extend
   `apply_structural_plan`/`compose_outputs` so decoration edits compose with retirement edits, moves, and status edits
   from stable source spans.
7. **Output contract.** Add the human marker section/summary wording and the two JSON lists; include marker edits in the
   no-op decision.
8. **Docs.** Update `docs/mark-next-tasks.md` (new "Pomodoro marker" section with the grammar table and examples,
   updated JSON schema and relocation notes) and the README summary.

## Tests and verification

### Plugin tests (`scripts/test-task-status-cycler.cjs`, run `npm test` and `npm run validate`)

- Pure helper coverage: marking plain/embedded/struck/aliased tokens, multiple links per bullet, prose prefixes,
  tabs/spaces, CRLF, fenced protection, idempotent re-marking, duplicate-marker collapse, marker stripping.
- Completion flow: originals marked (including already-struck and embedded bullets), copies unmarked, placeholder and
  cursor behavior unchanged, empty-child fallback unchanged, re-completing a reopened Pomodoro with marked bullets
  copies clean bullets forward.
- Retirement: preserves markers when striking; adds the marker when striking under a done Pomodoro; never marks under
  open Pomodoros or `#task` ancestry; active-editor and vault-backed notes both covered.
- Navigation regression (`scripts/test-navigation-hotkeys.cjs`): a 🍅-prefixed bullet is not parsed or reconciled as a
  managed dependency bullet.
- Deploy with `bob plugins sync` and manually exercise `<Ctrl+Enter>` completion + task closure in a scratch daily note,
  confirming marker placement, copy cleanliness, and strike-with-marker rendering in live preview.

### CLI tests (`just all`; unit tests in `mark_next.rs`, integration in `tests/cli.rs` + `tests/fixtures/mark_next/`)

- Occurrence parsing: marked/unmarked detection, marker-with-alias, marker before embeds, duplicate markers,
  adjacent-tilde spacing rules.
- Planning: retirement under done vs open Pomodoros produces the right grammar; marker-only repairs (add under done,
  remove under open) for live, struck, and embedded tokens; relocation to the completed fallback marks while relocation
  to the current Pomodoro does not; mixed bullets touch only link tokens; cancelled Pomodoros untouched.
- Contract: dry-run writes nothing and reports would-mark; JSON lists populated and stable; human wording; no-op run
  prints `already in sync` only when no status, retirement, move, or marker edits remain; second run is a no-op; CRLF
  preserved.
- Update the daily fixture with a done Pomodoro holding unmarked live and struck links (historical repair case) and
  refresh integration assertions.
- `collect_done` regression: archive-move link repair on a marked daily-note link preserves the marker.
- Finish with `bob mark-next-tasks --dry-run --format json` against the live vault, review the proposed marker repairs,
  apply once after review, and confirm the following dry run reports already in sync.

## Expected files

### `bob-plugins`

- `plugins/task-status-cycler/main.js`
- `plugins/task-status-cycler/manifest.json`
- `scripts/test-task-status-cycler.cjs`
- `scripts/test-navigation-hotkeys.cjs` (regression only)
- `README.md`

### `bob-cli`

- `src/native/mark_next.rs`
- `tests/cli.rs`
- `tests/fixtures/mark_next/2026/20260710.md`
- `tests/` collect_done regression (existing test module in `src/native/collect_done.rs` or `tests/cli.rs`)
- `docs/mark-next-tasks.md`
- `README.md`

## Sequencing and risk controls

1. Land the pure marker helpers with tests in both repos first (plugin helpers; occurrence-model extension).
2. Wire the plugin completion plan and retirement paths; run the full plugin suite; deploy and manually verify.
3. Wire the CLI planning/apply/output changes; run `just all`.
4. Live-vault dry run, diff review, single apply, idempotence check.

Top risks and their mitigations: marking copies instead of originals (copies always built from marker-stripped source
text, covered by completion tests); double-marking or marker-inside-strike output (single tolerant detector shared by
all writers, idempotence tests on both sides); disturbing dependency bullets or `#task` retirement output (marker gated
on Pomodoros-section ancestry + done checkbox state, regression tests); and destabilizing the `mark-next` no-op contract
(marker edits included in the sync decision with explicit second-run assertions).
