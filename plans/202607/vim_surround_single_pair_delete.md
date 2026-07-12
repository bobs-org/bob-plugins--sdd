---
create_time: 2026-07-12 12:24:02
status: done
prompt: 202607/prompts/vim_surround_single_pair_delete.md
tier: tale
---

# Plan: Delete One Repeated Vim-Surround Pair at a Time

## Context and intended behavior

`bob-vim-surround` currently discovers repeated symmetric delimiters such as `~~content~~` and `***content***` as
equal-length pairs of maximal same-line runs. That discovery fixed ambiguity between a repeated Markdown-style boundary
and several adjacent empty single-character pairs, but the shared edit-span builder makes `ds` delete the entire matched
run on both sides. As a result, one `ds~` changes `~~content~~` directly to `content`, and one `ds*` changes
`***content***` directly to `content`.

Retain maximal-run discovery, but make each delete-surround command remove exactly one symmetric delimiter character
from the opening run and one from the closing run. The visible behavior should peel repeated boundaries one layer at a
time:

- `ds~` changes `~~content~~` to `~content~`;
- `ds*` changes `***content***` to `**content**`;
- another `ds` (or one dot-repeat) removes exactly one more pair;
- a single-character pair still disappears in one command.

This change is deliberately limited to deletion. `cs` should continue replacing the complete repeated target run with
one requested replacement pair, because change-surround is a single replacement operation rather than incremental
peeling. `ys`, structural bracket matching, bracket-alias padding removal, accepted key handling, same-line sequential
run pairing, equal-length validation, malformed-run no-ops, event consumption, and recorded-action semantics should
remain unchanged.

## Implementation

1. Open the `bob-plugins` linked repository through the required `sase workspace open` flow and make all source changes
   in the returned source-of-truth checkout.

2. In `plugins/bob-vim-surround/main.js`, separate delete-span construction from the whole-boundary spans used by
   change-surround. For a repeated symmetric match, construct two one-code-unit deletion ranges—one within the matched
   opening run and one within the matched closing run—while preserving the current right-to-left edit application and
   cursor placement at the matched opening boundary. For a one-character symmetric match, retain the existing result.
   Keep structural pairs on their current path so `ds(`, `ds[`, and their aliases still remove their single delimiter
   characters plus only the padding whitespace already covered by the existing policy.

3. Leave symmetric run discovery and the match representation intact. In particular, continue pairing maximal runs
   sequentially, requiring equal opening and closing lengths, accepting the cursor anywhere in either matched boundary,
   and returning no match for unpaired or unequal runs. This lets the remaining equal-length runs be found again by a
   subsequent `ds` or dot-repeat without reintroducing the original repeated-delimiter discovery bug.

4. Update `scripts/test-vim-surround.cjs` with focused regressions that prove the command granularity rather than only
   the final edit count:
   - change double and triple symmetric deletion expectations to leave one fewer delimiter on each side, including the
     reported block-link form;
   - assert that each `ds` issues exactly two single-character delimiter deletions for repeated symmetric runs and keeps
     the established cursor and action-recording behavior;
   - cover representative cursor positions in the content and across both opening and closing runs;
   - exercise consecutive direct deletes and consecutive dot-repeats on the same triple-delimited surround, verifying
     the progression from three delimiters to two, then one, then none, with one pair removed per invocation;
   - preserve coverage showing that dot-repeat can apply the same one-pair deletion to another repeated surround;
   - retain `cs` expectations that complete repeated runs become one replacement pair, including padded structural
     replacements;
   - retain safe no-ops for odd, unpaired, and unequal runs and existing coverage for single-character symmetric pairs,
     bracket aliases/padding, rejected input, and normal-mode event behavior.

5. Release this user-visible patch as `bob-vim-surround` version `1.5.2`. Update
   `plugins/bob-vim-surround/manifest.json` and the root `README.md` version references, and clarify the documented
   repeated-run policy: discovery treats equal-length maximal runs as paired boundaries, `cs` replaces whole runs, and
   `ds` removes one delimiter from each side per command.

## Validation and deployment

1. Run `node --test scripts/test-vim-surround.cjs` while iterating, then run the complete `npm test` suite and
   `npm run validate` from `bob-plugins` to cover repository regressions, syntax, and manifest consistency.
2. Review `git diff --check`, the final diff, version references, and repository status. Confirm the change is limited
   to the surround implementation, focused tests, manifest, and documentation, and that the linked repository began
   clean.
3. Deploy with a non-forced `bob plugins sync -p bob-vim-surround -r <linked-bob-plugins-path> --no-pull` using the
   tested linked checkout. Stop rather than force an overwrite if the vault reports or exhibits a conflict. Because the
   prior deployment was externally reverted after a successful copy, verify `manifest.json` and `main.js` against the
   source byte-for-byte immediately after sync and again after a short follow-up check; report any external rewrite as a
   deployment blocker while preserving the validated source changes.
