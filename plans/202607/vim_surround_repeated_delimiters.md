---
create_time: 2026-07-12 12:09:50
status: done
prompt: 202607/prompts/vim_surround_repeated_delimiters.md
tier: tale
---

# Plan: Support Repeated Symmetric Vim-Surround Delimiters

## Context and root cause

The `bob-vim-surround` plugin in the linked `bob-plugins` repository accepts visible one-code-unit characters such as
`~`, `*`, and `x` as symmetric surround targets. Its symmetric-pair finder currently gathers individual occurrences on
the cursor's line and pairs them strictly by ordinal position: occurrence 1 with 2, occurrence 3 with 4, and so on. That
works for separate single-character pairs such as `~one~ ~two~`, but it misinterprets repeated Markdown-style
boundaries. In `~~[[sase#^fix-sdd-error]]~~`, the four tildes become two adjacent empty pairs (`~~` and `~~`), so no
pair encloses a cursor anywhere in the block link. `cs~` and `ds~` consume their input normally, but discovery returns
no match and the edit layer has nothing to apply.

Treat a maximal contiguous run of the same symmetric delimiter as one boundary token. Two sequential same-line runs with
the same length form a candidate surround, so `~~content~~` and `***content***` are well-defined while separate pairs
remain independent. `ds` should remove both complete boundary runs, and `cs` should replace each complete run once with
the requested replacement pair. Thus `ds~` turns `~~[[sase#^fix-sdd-error]]~~` into `[[sase#^fix-sdd-error]]`, while
`cs~x` turns it into `x[[sase#^fix-sdd-error]]x`. Requiring equal run lengths avoids silently rewriting malformed or
ambiguous input such as `~~content~`; unmatched or mismatched runs should remain a safe no-op. Delimiter occurrences
inside content remain inherently ambiguous, and matching remains intentionally same-line and non-nesting.

## Implementation

1. Open the `bob-plugins` linked repository through the required `sase workspace open` flow for the current numbered
   primary workspace, and make all implementation changes in the returned source-of-truth path.

2. In `plugins/bob-vim-surround/main.js`, revise symmetric-delimiter discovery to collect maximal contiguous runs rather
   than individual character offsets, pair those runs sequentially on the current line, require matching run lengths,
   and return the enclosing pair for the cursor. Preserve the existing deterministic behavior for separate
   single-character pairs, cursor inclusion at either boundary, the accepted-character policy, and the same-line
   ambiguity limitation.

3. Generalize the internal surround match/edit representation so opening and closing boundaries can each span one or
   more characters. Make `buildChangeSurroundEdit` replace an entire matched run with one replacement boundary and make
   `buildDeleteSurroundEdit` delete the entire run. Keep bracket matching as single-character structural ranges, retain
   opening-alias whitespace trimming and replacement padding, preserve operation ordering and cursor placement, and
   leave `ys` behavior unchanged because it creates the explicitly requested single replacement pair.

4. Extend `scripts/test-vim-surround.cjs` with focused, table-driven regressions covering:
   - the reported `~~[[sase#^fix-sdd-error]]~~` case at representative cursor positions, including exact `ds~` output;
   - `cs` replacing double and triple symmetric runs exactly once on each side, including a structural/padded
     replacement;
   - `ds` removing complete double and triple runs without leaving partial delimiters;
   - continued single-character behavior and deterministic selection among multiple same-line pairs of delimiter runs;
   - cursor positions on the opening and closing runs, plus safe no-ops for odd, unpaired, or unequal-length runs;
   - dot-repeat replaying `cs` and `ds` against repeated target runs while preserving the existing recorded action and
     cursor behavior;
   - unchanged bracket aliases, padding, rejected key input, event consumption, and ordinary normal-mode behavior.

5. Release the backward-compatible bug fix as `bob-vim-surround` version `1.5.1`. Update
   `plugins/bob-vim-surround/manifest.json` and the root `README.md`, replacing the current statement about sequential
   character pairs with the repeated-run matching policy and its equal-length/same-line ambiguity limits.

## Validation and deployment

1. Run `node --test scripts/test-vim-surround.cjs` while iterating, then run the complete `npm test` suite and
   `npm run validate` from `bob-plugins` to cover repository regressions, syntax, and manifest consistency.
2. Review `git diff --check`, the final diff, version references, and repository status to confirm that the change is
   limited to the surround implementation, focused tests, manifest, and documentation, with no unrelated workspace
   changes.
3. Deploy with a non-forced `bob plugins sync -p bob-vim-surround -r <linked-bob-plugins-path> --no-pull` so the sync
   uses the tested numbered linked workspace rather than the configured canonical checkout. Stop and report any dirty
   vault conflict instead of forcing an overwrite, then verify the deployed `manifest.json` and `main.js` are
   byte-for-byte identical to the source files.
