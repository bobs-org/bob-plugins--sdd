---
create_time: 2026-07-12 08:59:45
status: done
prompt: .sase/sdd/plans/202607/prompts/restore_pomodoro_and_tab_pin_keymaps.md
tier: tale
---

# Plan: Restore `\p` for Pomodoro increments and move tab pinning to `\s`

## Goal

Restore the established Obsidian Vim normal-mode `\p` behavior that increments the current Pomodoro time range, while
retaining the new **Toggle current tab pin** command and assigning its Vim trigger to `\s`. Both mappings must remain
deterministic regardless of plugin load order, preserve their intended Vim-count behavior, and be deployed from the
`bob-plugins` source-of-truth repository rather than edited in the vault.

## Current state and key decision

- Bob Navigation Hotkeys owns the tab-management command and currently maps `\p` to its count-independent
  `bobNavigationToggleCurrentTabPin` Vim action.
- Bob Ledger Tools previously owned `\p` through `bobLedgerAddPomodoroUnit`, applying the Vim repeat count as a positive
  unit delta. The previous tab-pin change removed both that action definition and its mapping, while leaving `\P`, `\o`,
  and `\O` intact.
- No plugin in the repository currently claims `\s`, so moving the navigation mapping there avoids another known
  cross-plugin collision.

Keep ownership aligned with behavior: Bob Ledger Tools will exclusively own `\p`, and Bob Navigation Hotkeys will
exclusively own `\s`. Do not duplicate either mapping across plugins or change the existing tab-pin command, active-leaf
resolution, delayed Vim-adapter registration, or idempotence logic.

## Implementation

1. Restore the positive Pomodoro Vim action in Bob Ledger Tools.
   - Reintroduce `bobLedgerAddPomodoroUnit` alongside the existing subtract and range-offset actions.
   - Delegate to the existing `changePomodoroUnits` implementation with a positive `getVimRepeat(actionArgs)` value, so
     bare `\p` adds one unit and a count such as `3\p` adds three units.
   - Map `\p` to that action in normal mode.
   - Preserve `\P` subtraction and `\o`/`\O` range movement exactly as they are.

2. Move only the Bob Navigation Hotkeys Vim trigger from `\p` to `\s`.
   - Keep the action name, command registration, `toggleCurrentTabPin()` behavior, supported
     `WorkspaceLeaf.togglePinned()` API usage, active-leaf-at-invocation semantics, safe failure behavior, layout-ready
     retry, and idempotent registration unchanged.
   - Continue ignoring Vim repeat counts for the tab toggle so a count cannot toggle multiple times and cancel itself
     out.

3. Update plugin metadata and repository documentation.
   - Patch-bump Bob Navigation Hotkeys from `1.13.0` to `1.13.1` for the corrected tab-pin trigger.
   - Patch-bump Bob Ledger Tools from `1.1.0` to `1.1.1` for restoring the accidentally removed mapping.
   - Synchronize the root README version table with both manifests.
   - Replace the README test description that treats `\p` as transferred away from Ledger Tools with wording that
     documents the distinct `\p` Pomodoro and `\s` tab-pin ownership.
   - Keep the existing manifest descriptions unless implementation reveals they are inaccurate; both already describe
     their plugins' behavior without hard-coding the old trigger.

## Tests and validation

1. Update the focused Bob Navigation Hotkeys tests to verify:
   - the tab-pin action maps exactly once to `\s` in normal mode and does not claim `\p`;
   - invoking the action toggles the active leaf exactly once even when Vim supplies a repeat count;
   - active-leaf resolution, safe failure cases, delayed adapter availability, and registration idempotence continue to
     pass unchanged.

2. Update the focused Bob Ledger Tools Vim-mapping test to verify:
   - `bobLedgerAddPomodoroUnit` is defined and `\p` maps to it in normal mode;
   - invoking it passes a positive default unit and honors an explicit repeat count;
   - `\P`, `\o`, and `\O` still map to their existing actions in normal mode;
   - the Ledger mapping set does not claim `\s`.

3. Run the complete repository checks:
   - `npm test`
   - `npm run validate`
   - review the final diff and confirm only the two plugin implementations, their manifests, focused tests, and README
     changed.

4. Deploy and smoke-test both plugins.
   - Resolve the linked `bob-plugins` workspace through `sase workspace open`, then sync each changed plugin with
     `bob plugins sync --no-pull -r <resolved-linked-repo> -p <plugin-id>` so deployment uses the reviewed workspace
     contents rather than the canonical checkout. Do not edit deployed vault files directly and do not use `--force`
     unless separately authorized after a vault-local-change warning.
   - Verify each managed source file copied to the vault matches the linked-workspace version byte-for-byte.
   - Reload both plugins or restart Obsidian so the global CodeMirror Vim mappings are refreshed, then confirm:
     - `\p` adds one Pomodoro unit and a counted form adds the requested number of units;
     - `\P`, `\o`, and `\O` retain their current behavior;
     - successive `\s` presses pin and unpin the active tab, including after switching tabs;
     - `\p` never toggles a tab and `\s` never edits a Pomodoro line.

## Scope boundaries

- Do not change Obsidian hotkey JSON, `.obsidian.vimrc`, user Vim configuration, or unrelated plugin mappings.
- Do not rename the tab-pin command or Vim action, rework its lifecycle, or alter Pomodoro unit/range-editing semantics.
- Do not edit the vault's deployed plugin files directly; deploy only through `bob plugins sync` from the resolved
  linked source workspace.
- Do not commit implementation changes unless explicitly requested or triggered by the normal post-completion finalizer.
