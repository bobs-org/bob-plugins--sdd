---
create_time: 2026-07-12 08:09:42
status: done
prompt: .sase/sdd/plans/202607/prompts/toggle_obsidian_tab_pin.md
tier: tale
---

# Plan: Add a `\p` Vim keymap for toggling the current Obsidian tab pin

## Goal

Make `\p` in Obsidian Vim normal mode pin an unpinned active tab and unpin a pinned active tab. The behavior should be
available as a regular Bob Navigation Hotkeys command as well as through the Vim mapping, use Obsidian's supported
workspace-leaf API, and remain deterministic regardless of plugin load order.

## Current state and key decision

- Bob Navigation Hotkeys already owns the neighboring tab-management commands (move, duplicate, and close tabs), so it
  is the natural owner of the new pin-toggle command and Vim action.
- Bob Ledger Tools currently claims `\p` for adding one Pomodoro unit. Leaving that registration in place would make the
  requested behavior load-order dependent because two enabled plugins would map the same normal-mode sequence.
- The new tab behavior will therefore supersede the old `\p` Pomodoro binding. Remove only that mapping and its now-
  unused positive-unit Vim action from Bob Ledger Tools; retain `\P`, `\o`, and `\O` and retain the shared Pomodoro
  unit-editing implementation used by subtraction. Do not invent a replacement Pomodoro binding as part of this change.

## Implementation

1. Add a **Toggle current tab pin** command to Bob Navigation Hotkeys alongside its existing tab commands.
   - Resolve `app.workspace.activeLeaf` at invocation time so the command always targets the currently active tab.
   - Call the public `WorkspaceLeaf.togglePinned()` API, which lets Obsidian own the pin-state transition, layout
     persistence, and UI update.
   - Return cleanly when there is no active leaf or the leaf cannot be toggled, avoiding exceptions during startup or
     unusual workspace states.

2. Give Bob Navigation Hotkeys an idempotent Vim-registration lifecycle consistent with the repository's other Vim-
   aware plugins.
   - Register after layout readiness, and retry on an active-leaf change if the CodeMirror Vim adapter was not yet
     available.
   - Define a navigation-specific action that delegates to the same tab-toggle method as the command.
   - Map `\p` to that action in normal mode and ensure repeated registration attempts cannot install duplicates.
   - Keep the mapping count-independent: a Vim count must not toggle repeatedly and accidentally cancel itself out.

3. Remove Bob Ledger Tools' conflicting `\p` registration and its unused `bobLedgerAddPomodoroUnit` Vim action. Preserve
   all other ledger mappings and behavior so the change is limited to transferring ownership of this one key sequence.

4. Update plugin metadata and repository documentation.
   - Bump Bob Navigation Hotkeys for the new command/keymap and Bob Ledger Tools for the intentional binding change.
   - Synchronize their README table versions and descriptions with the manifests, documenting the tab-pin behavior and
     avoiding any implication that Ledger Tools still owns `\p`.

## Tests and validation

1. Extend the Bob Navigation Hotkeys Node tests with a fake CodeMirror Vim adapter and workspace leaves to verify:
   - `\p` is registered exactly once as a normal-mode action;
   - invoking the action calls `togglePinned()` exactly once on the active leaf;
   - invoking the command method after the active leaf changes targets the new leaf;
   - missing workspace, missing active leaf, or a leaf without the supported API fails safely;
   - delayed adapter availability can retry and successful registration remains idempotent.

2. Add focused Bob Ledger Tools mapping coverage and include it in `npm test` to verify that `\p` is no longer
   registered while `\P`, `\o`, and `\O` retain their existing actions and normal-mode context. This is the regression
   guard against reintroducing the cross-plugin collision.

3. Run the repository checks:
   - `npm test`
   - `npm run validate`

4. Deploy both changed source-of-truth plugins to the Bob vault with targeted `bob plugins sync -p ...` commands, as
   required by the linked repository instructions. Then perform a live Obsidian smoke test after reloading the plugins:
   - open multiple tabs, enter Vim normal mode, and confirm successive `\p` presses visibly pin then unpin the active
     tab;
   - switch tabs and confirm the newly active tab is the target;
   - confirm `\p` no longer edits a Pomodoro line and the retained Ledger Tools mappings still work.

## Scope boundaries

- Do not edit deployed plugin files directly in the vault; all source changes belong in the `bob-plugins` linked
  repository and reach the vault only through `bob plugins sync`.
- Do not modify Obsidian hotkey JSON, Vim configuration files, or unrelated tab commands.
- Do not assign a speculative replacement for the retired Pomodoro `\p` binding; that can be chosen separately if
  desired.
