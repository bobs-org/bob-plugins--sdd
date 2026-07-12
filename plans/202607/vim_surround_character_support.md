---
create_time: 2026-07-12 11:55:14
status: done
prompt: 202607/prompts/vim_surround_character_support.md
tier: tale
---

# Plan: Broaden Vim-Surround Character Support

## Context and intended behavior

The `bob-vim-surround` plugin in the linked `bob-plugins` repository owns the `ys`, `cs`, and `ds` behavior. Its
pair-resolution helper currently preserves vim-surround-style bracket aliases and accepts arbitrary punctuation as
symmetric delimiters, but it explicitly rejects letters and numbers. As a result, `~` and `^` are nominally supported
already while `x` is not, and none of that broad fallback behavior has focused automated coverage.

Adopt one shared character policy for all three operations:

- Keep the existing structural pairs and padding rules: opening `(`, `[`, `{`, and `<` select padded matching pairs,
  while their closing counterparts select the same pairs without padding.
- Treat every other visible, single-UTF-16-code-unit letter, number, punctuation character, or symbol as a literal
  symmetric delimiter. This includes every printable non-space ASCII key (`!` through `~`), so examples such as `~`,
  `x`, `^`, digits, uppercase letters, `s`, `.`, and `\\` work without a per-character allowlist.
- Continue rejecting whitespace, control/navigation/modifier keys, modifier chords, dead-key/IME composition, combining
  marks, and multi-code-unit characters such as emoji. These inputs are either not visible delimiters, can collide with
  editor/browser commands, or do not fit the plugin's current one-code-unit matching and range model.
- Preserve the current symmetric-delimiter discovery semantics: `cs` and `ds` identify a same-line pair around the
  cursor. Arbitrary letters are inherently more likely than punctuation to occur inside their own delimited content, so
  tests and documentation should make this deterministic limitation clear rather than implying nested parsing that the
  format cannot provide.

## Implementation

1. Open the `bob-plugins` linked repository through the required `sase workspace open` flow for the current numbered
   primary workspace, and make all plugin changes in that returned source-of-truth path.

2. In `plugins/bob-vim-surround/main.js`, replace the punctuation-only fallback classification with the shared visible
   single-character policy above. Route pair lookup for `ys` additions, both `cs` stages (target and replacement), and
   `ds` targets through that same policy, retaining the explicit bracket-pair table and its padding behavior. Keep the
   pending-input ordering that lets normally meaningful Vim keys such as `x`, `s`, `c`, `d`, `y`, and `.` act as
   delimiters only after a surround command has begun, so ordinary normal-mode commands and dot-repeat remain unaffected
   outside that state.

3. Add `scripts/test-vim-surround.cjs` using the repository's existing Node test/stub pattern and register it in the
   root `package.json` test command. Use table-driven coverage to verify:
   - every printable non-space ASCII character resolves to either its explicit structural pair or a literal symmetric
     pair, with representative visible BMP Unicode characters accepted;
   - whitespace, controls, composition/modifier input, combining marks, multi-character strings, and multi-code-unit
     characters are rejected without leaking text or invoking a surround edit;
   - `ys` can add `~`, `x`, `^`, alphanumeric, shifted-symbol, trigger-letter, dot, and backslash surrounds through the
     pending keydown path;
   - `cs` accepts broad characters independently in both target and replacement positions, finds the enclosing symmetric
     pair, edits both sides, and preserves the established bracket padding rules;
   - `ds` finds and removes the same representative broad character set;
   - accepted events are consumed exactly once, cancellation still clears pending state, cursor/edit behavior remains
     correct, and dot-repeat records/replays broad-character `ys`, `cs`, and `ds` actions without stealing a literal `.`
     entered while a surround is pending.

4. Treat the change as a backward-compatible feature release: increment `plugins/bob-vim-surround/manifest.json` from
   `1.4.0` to `1.5.0`, update the plugin version in the root `README.md`, and briefly document the accepted-character
   policy and exclusions so the supported surface is explicit rather than inferred from code.

## Validation and deployment

1. Run the focused vim-surround test directly while iterating, then run the complete `npm test` suite and
   `npm run validate` from `bob-plugins` to cover repository regressions, manifest consistency, and syntax validation.
2. Review the final diff to ensure changes are limited to the surround implementation, its focused tests/test
   registration, and version/documentation updates.
3. Run `bob plugins sync -p bob-vim-surround` as required by the linked repository instructions, and verify that the
   updated manifest and `main.js` deploy to the vault without requiring a forced overwrite. If sync reports dirty vault
   files, stop and report the conflict rather than overwriting them.
