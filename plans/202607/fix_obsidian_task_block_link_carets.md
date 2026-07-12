---
create_time: 2026-07-12 15:24:55
status: done
prompt: 202607/prompts/fix_obsidian_task_block_link_carets.md
tier: tale
---

# Plan: Make caret task-link completion reliable on ordinary Obsidian task sub-bullets

## Summary

Fix the `block-id-prompt` Obsidian plugin so the caret workflow reliably completes task block links from ordinary task
sub-bullets as well as Pomodoro sub-bullets. The task picker and block-ID completion remain available in both contexts,
while the selected target task is promoted to `[*]` (Next) only when the source link is beneath a Pomodoro entry.

The source-of-truth change belongs in the linked `bob-plugins` repository and must be deployed to the vault with
`bob plugins sync`; the generated plugin copy under the vault will not be edited directly.

## Diagnosis / root cause

The problem is not an intentional source-context restriction. `inspectActiveEditor` searches for caret markers before it
checks whether the source is a Pomodoro child, and the Pomodoro predicate is consulted only after a task is selected to
decide whether to promote `[ ]` to `[*]`.

The actual failure is a timing hole between the two caret stages:

1. A single caret typed after a wiki link produces `[[target]]^`.
2. After the plugin's 75 ms document-change debounce, the file-link-jump parser rewrites that as `[[target^]]` and
   places the cursor inside the closing brackets.
3. A second caret then produces `[[target^^]]`, which the task-picker parser recognizes.

If both carets are typed before the first debounced scan runs, however, the editor contains `[[target]]^^`. The
single-caret parser deliberately rejects it because the trailing marker is not a single caret, while the task-picker
parser only recognizes carets already inside the wiki-link destination. The state therefore matches neither parser and
the command silently does nothing. This affects ordinary and Pomodoro sub-bullets alike, but is easy to encounter in the
ordinary task workflow when `^^` is typed as one quick sequence.

The existing `scripts/test-block-id-prompt.cjs` suite covers dependency identity collection only. It has no cases for
the single-caret relocation, rapid double-caret state, source-line context, or the Pomodoro-only Next-status rule, which
allowed the race to remain unnoticed.

## Implementation

### 1. Recognize the rapid two-caret form atomically

Extend the marker-discovery layer in `plugins/block-id-prompt/main.js` to recognize two adjacent carets immediately
after an eligible wiki link (`[[target]]^^`, including embedded links and same-file/block-link variants) as the same
logical task-picker request produced by the existing two-stage flow.

The direct form should normalize through the same destination rules as the staged form:

- a plain destination behaves like `[[target^]]` followed by the second caret;
- an existing `#^block-id` is reset to the block-selection position before opening the picker;
- same-file `[[#^block-id]]` links resolve against the active note;
- source ranges include the two external carets so completion or cancellation cannot leave marker residue;
- cursor proximity, fenced-code rejection, alias handling, and embedded-link preservation remain consistent with the
  current staged behavior.

Keep the existing single-caret relocation path intact for users who intentionally type one caret and pause. Prefer a
small shared normalization/parser helper so the staged and rapid forms cannot drift.

### 2. Preserve context-specific task status behavior

Do not broaden `shouldPromoteTaskToNext`. Completing a task link from an ordinary Obsidian task sub-bullet must:

- open the same task picker;
- reuse an existing block ID or prompt for a new one;
- finish the source as a canonical block link;
- leave the selected task's checkbox status unchanged.

Completing from a Pomodoro descendant continues to promote only a currently open `[ ]` task to `[*]`. In-progress `[/]`,
already-Next `[*]`, and other statuses remain unchanged. Both existing-ID and new-ID completion paths retain their
current snapshot/staleness guards.

### 3. Add focused regression coverage

Expand `scripts/test-block-id-prompt.cjs` and the plugin's exported test helpers as narrowly as needed to cover:

- the existing single-caret relocation form and its cursor/replacement coordinates;
- rapid `[[target]]^^` recognition on a representative ordinary task child bullet;
- embedded, same-file, existing-block-ID, and canonical `#^` variants;
- equivalence between the rapid form and the staged inside-link form;
- cancellation/revert data and full source ranges, ensuring no external caret is left behind;
- rejection outside the cursor window and inside fenced code where applicable;
- ordinary task descendants do not request Next promotion;
- Pomodoro descendants promote `[ ]` only, while `[/]` and `[*]` remain untouched.

Where practical, exercise marker discovery and promotion as pure helpers rather than relying on timers in tests. Add a
small plugin-level fake-editor test only if needed to prove the final replacement/cursor behavior.

### 4. Deploy the source-of-truth plugin

After implementation and validation, run `bob plugins sync` from the linked `bob-plugins` checkout for
`block-id-prompt`, first in dry-run mode and then for real, as required by that repository. Confirm the deployed vault
copy is byte-for-byte identical to the linked source. The user may need to reload the plugin or Obsidian for the new
JavaScript to become active.

## Verification

Run:

- `node -c plugins/block-id-prompt/main.js`
- `npm test`
- `npm run validate`
- `git diff --check`
- `bob plugins sync -p block-id-prompt -r "$PWD" --dry-run`, followed by the real sync
- syntax and byte-comparison checks against the deployed vault copy

Then manually smoke-test in Obsidian:

1. Beneath an ordinary `[ ] #task`, create a child wiki link and type `^^` quickly. The picker opens, the chosen task is
   linked (with a new ID prompt if needed), and its status does not change.
2. Repeat with a deliberate pause after the first caret. The first caret moves inside the link and the second opens the
   same picker.
3. Repeat beneath a Pomodoro entry. An open selected task becomes `[*]`; an in-progress selected task stays `[/]`.
4. Repeat with a same-file link and an embedded link.
5. Confirm caret-like text in fenced code and unrelated prose remains untouched.

## Scope boundaries

- No changes to `bob-cli`, task status definitions, Tasks plugin configuration, or task-status styling.
- No automatic `dependsOn`/transclusion conversion beyond the existing block-link completion behavior.
- No edits to vault notes or the generated vault plugin source by hand.
- No change to which `#task` statuses appear in the task picker.
