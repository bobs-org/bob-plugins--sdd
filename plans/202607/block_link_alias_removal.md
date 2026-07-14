---
create_time: 2026-07-14 08:30:34
status: done
prompt: 202607/prompts/block_link_alias_removal.md
tier: tale
---

# Plan: Restore alias removal in Obsidian block-link caret completion

## Summary

Fix the `block-id-prompt` Obsidian plugin so typing `^` immediately after a wiki/block link removes any display alias,
normalizes an existing block target to the block-selection position, inserts the caret immediately before the link's
closing `]]`, and leaves the cursor after that inserted caret. Keep the rapid `^^` task-picker path behaviorally
equivalent to typing the two carets with a pause between them.

The source-of-truth implementation is in the linked `bob-plugins` repository. After validation, deploy only the
`block-id-prompt` plugin through `bob plugins sync`; do not edit the vault's generated plugin copy directly.

## Diagnosis / root cause

The editor mutation and cursor arithmetic are working as designed. `parseTrailingCaretCompletionMarker` computes the
alias-free insertion coordinate from the destination portion of the wiki link, and both the CodeMirror dispatch path and
the fallback editor-API path use that coordinate to place the caret before the first closing bracket.

The alias is retained because commit `002824e` (`fix(block-id-prompt): handle rapid task picker carets`) changed the
marker's replacement strings from alias-free forms to forms that append `aliasSuffix`:

- `plainReplacement` now reconstructs the link with the alias before the fallback path inserts `^`;
- `completionReplacement` now reconstructs the final staged link with the alias in the CodeMirror transaction;
- the rapid external `^^` parser also carries `aliasSuffix` into task-picker completion and cancellation, bypassing the
  single-caret normalization step entirely.

The same commit added a regression test named `single caret relocation preserves embedding and aliases`, so the test
suite currently passes while explicitly asserting the unwanted result. The earlier plan requested consistent alias
handling but did not require alias preservation; the commit encoded that mistaken interpretation in both production code
and tests.

## Implementation

### 1. Make single-caret normalization alias-free

Update the file-link-jump marker produced by `parseTrailingCaretCompletionMarker` in `plugins/block-id-prompt/main.js`
so both replacement forms omit the parsed display alias. Preserve the existing destination normalization rules:

- a plain aliased link such as `[[Projects|Work queue]]^` becomes `[[Projects^]]`;
- an aliased block link such as `[[Projects#^existing|Ship it]]^` resets to `[[Projects#^]]`;
- a leading embed marker (`!`) remains untouched because it is outside the replaced wiki-link range;
- the cursor lands immediately after the inserted `^`, before the first `]`.

Keep the current atomic CodeMirror transaction and editor-API fallback. They should consume the same alias-free marker
data so runtime behavior cannot differ by editor path.

### 2. Keep rapid `^^` behavior equivalent

Normalize aliases out of `parseRapidTaskPickerMarker` as well. Rapid input such as `[[Projects|Work queue]]^^` must
produce the same task-picker source, completion result, and cancellation/revert state as the staged sequence
`[[Projects|Work queue]]^` followed by a pause and another `^`. Retain embedding, same-note targets, existing-block
reset behavior, full source ranges, cursor proximity checks, and fenced-code protection.

Do not globally remove aliases from unrelated block-link operations such as explicit block-ID rename/reference rewrites;
scope the behavior to caret-triggered link completion.

### 3. Correct and extend regression coverage

Update `scripts/test-block-id-prompt.cjs` to replace the alias-preservation expectation with the intended contract and
cover both parser output and final editor state. Include:

- a plain aliased embedded link, proving the alias is removed while `!` is preserved;
- an aliased link with an existing `#^block-id`, proving the old block ID and alias are removed before the new caret is
  staged and the cursor coordinate remains correct after the shorter replacement;
- a rapid aliased `^^` input, proving its marker/revert data no longer carries the alias;
- an end-to-end task-picker completion case from rapid aliased input, proving the final canonical block link is
  alias-free.

Retain the existing non-aliased, same-file, block-link, cancellation-range, cursor-proximity, fenced-code, and Pomodoro
status tests to guard against regressions in the rapid-caret fix.

### 4. Validate and deploy

Run focused and repository-wide checks in `bob-plugins`:

- `node --check plugins/block-id-prompt/main.js`
- `node --test scripts/test-block-id-prompt.cjs`
- `npm test`
- `npm run validate`
- `git diff --check`

Then preview and perform a source-of-truth deployment of `block-id-prompt` with `bob plugins sync`, as required by the
linked repository, and verify the deployed `main.js`/manifest/styles match the linked plugin files. Report if Obsidian
needs a plugin reload for the JavaScript change to become active.

## Scope boundaries

- No changes to `bob-cli`, task status behavior, picker eligibility, dependency handling, or block-ID rename semantics.
- No hand edits to vault notes or to the deployed plugin copy under the vault.
- No changes to aliases on ordinary links unless the caret completion workflow is triggered immediately after them.
