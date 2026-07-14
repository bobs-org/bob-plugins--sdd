---
create_time: 2026-07-14 09:29:49
status: done
prompt: 202607/prompts/dependency_status_propagation.md
tier: tale
---

# Dependency Status Propagation Plan

## Goal

Keep task dependency status consistent when a sole child block link is transcluded. The Bob Navigation Hotkeys `!`
action and `bob mark-next-tasks` should both promote the linked task to the parent's effective status within the ordered
lifecycle `ready [ ] < next [*] < in-progress [/]`. Promotion must be monotonic: never lower a task, never reinterpret
done/cancelled/custom statuses as part of this ordering, and never demote a target when a transclusion is removed.

This work spans the `bob-cli` repository and the linked `bob-plugins` source repository. The plugin source remains the
authoritative copy; the finished plugin change must be deployed to the vault with `bob plugins sync` only after its
tests pass.

## Current Behavior and Constraints

- Bob Navigation Hotkeys already treats a sole transcluded block-link child bullet as a task dependency. Adding `!`
  synchronizes `[dependsOn::]`, ensures the target's path-qualified `[id::]`, and supports same-note, cross-note, and
  counted toggles, but it does not change the target checkbox status.
- The plugin deliberately leaves arbitrary links, invalid/non-task endpoints, fenced examples, retired struck dependency
  links, and `#hide` targets outside dependency synchronization. Status promotion must use those same eligibility rules
  rather than broadening what `!` manages.
- `bob mark-next-tasks` resolves the same sole-transclusion graph recursively from tasks linked under open Pomodoros. It
  currently represents reachability as a set, promotes every reachable ready task only to Next, preserves every
  in-progress task, and clears unreachable Next tasks back to Ready.
- The command's existing link retirement, duplicate-Pomodoro pruning, note/block resolution, exclusion rules, dry-run
  guarantees, and unreachable-Next clearing policy are independent behaviors and must remain intact.
- Standard checkbox symbols are the supported ordered states for this feature: space is Ready, `*` is Next, and `/` is
  In Progress. Other configured or custom statuses are incomparable and remain unchanged.

## Status Semantics

Use one shared conceptual rule in both implementations:

| Parent/effective status | Target status                             | Result                        |
| ----------------------- | ----------------------------------------- | ----------------------------- |
| Next                    | Ready                                     | promote target to Next        |
| In Progress             | Ready or Next                             | promote target to In Progress |
| Next                    | In Progress                               | leave target In Progress      |
| Ready                   | Ready, Next, or In Progress               | no promotion                  |
| Any ranked parent       | done, cancelled, or unknown/custom target | leave target unchanged        |

Removing `!` only removes dependency metadata as it does today; it does not roll the target status back. When several
parents reach the same task, the strongest requested status wins. Recursive propagation uses the target's effective
status after considering both its current state and incoming requests, so an already-in-progress intermediate task also
promotes its lower-status descendants to In Progress. Cycles converge by reprocessing a node only when its effective
rank increases.

For `mark-next-tasks`, each surviving direct Pomodoro task retains the command's existing minimum desired state of Next.
A direct task that is already In Progress seeds In Progress instead. Each dependency then inherits the source task's
effective ranked state. This preserves the current meaning of an open-Pomodoro link while adding status-aware
propagation for dependency bullets created manually or through `!`.

## Implementation Work

### 1. Add monotonic promotion to Bob Navigation Hotkeys

In the linked `bob-plugins` repository:

- Add small, testable helpers in `plugins/bob-navigation-hotkeys/main.js` to read a real Obsidian task's checkbox
  status, compare the three supported ranks, and rewrite only the checkbox character when the target is lower than the
  parent.
- Extend the existing dependency-aware transclusion transaction, not the generic link toggler, so promotion happens only
  when a valid dependency changes from plain to transcluded. Derive the desired status from the validated parent task
  and leave untransclusion status-neutral.
- Apply same-file status changes in the editor content and cross-file changes inside the same guarded vault update that
  ensures the target `[id::]`. Preserve the current source snapshot checks and target-line revalidation so a concurrent
  edit cannot produce a partial or stale promotion.
- Aggregate repeated actions for one target and retain the strongest requested rank. Continue applying each valid
  parent's dependency metadata update even when multiple parents share a target.
- Keep the pure `planSameFileDependencyToggle` path behaviorally aligned with the runtime path, including no promotion
  for hidden targets, invalid endpoints, removal operations, or incomparable target statuses. Counted `N!` toggles
  should inherit the same logic without a separate status implementation.
- Export only the narrow helpers needed by the Node test harness. Bump the navigation plugin patch version in its
  manifest and update the plugin table/description in `README.md` to mention monotonic dependency status promotion.

### 2. Make `mark-next-tasks` propagate desired status ranks

In `src/native/mark_next.rs`:

- Replace the unlabelled dependency closure with a map from resolved `(note path, block ID)` identity to desired ranked
  status. Seed direct identities at Next or their stronger current In-Progress state, then run a monotonic, cycle-safe
  work queue across the existing resolved dependency edges.
- When propagating through a target, combine the incoming rank with the target's current supported rank before visiting
  its descendants. Merge multiple paths with the maximum rank. Preserve the current resolution warnings and
  duplicate-fragment behavior.
- Expand the transition model to support Ready-to-In-Progress and Next-to-In-Progress promotions while retaining the
  existing Ready-to-Next promotion, unreachable Next-to-Ready clearing, kept counters, and no-op handling for all
  terminal/custom statuses.
- Keep direct-versus-dependency attribution based on the final surviving Pomodoro references, so structural daily-note
  edits still determine which graph is active and directly linked tasks are not mislabeled as dependency-only.
- Add a `marked_in_progress` collection to the structured result rather than reporting `[*] -> [/]` or `[ ] -> [/]` as
  `marked_next`. Include it in change detection, human dry-run/apply sections, summaries, and JSON output while
  retaining all existing fields and meanings.

### 3. Update command-facing documentation

- Revise the `mark-next-tasks` long help and top-level command summary/examples in `src/native/mark_next.rs` and
  `src/runner.rs` so they describe status-aware dependency propagation without implying that every active dependency
  only becomes Next.
- Update the overview in `README.md` and the full contract in `docs/mark-next-tasks.md` with the ordered status table,
  recursive strongest-status rule, non-demotion guarantee, interaction with unreachable-Next clearing, and the new
  human/JSON reporting field.
- Keep the CLI syntax unchanged; this feature adds no subcommand or option.

## Test Plan

### Bob Navigation Hotkeys

Extend `scripts/test-navigation-hotkeys.cjs` to cover:

- same-file Ready-to-Next, Ready-to-In-Progress, and Next-to-In-Progress promotions;
- no demotion when the target is already stronger, when `!` is removed, or when the target is done/cancelled/custom;
- cross-file promotion combined atomically with `[id::]` creation and parent `[dependsOn::]` updates;
- counted toggles with different parent ranks, repeated targets where the strongest parent wins, and independent
  hidden/invalid targets;
- stale source or target snapshots leaving status and dependency metadata untouched rather than partially applied.

Run `npm test` and `npm run validate` in the linked plugin repository. After both pass and the source version is
updated, run `bob plugins sync` to deploy the authoritative plugin files to the vault.

### `bob mark-next-tasks`

Add focused Rust unit tests for the rank comparison, transition matrix, and labelled graph propagation, including
multiple parents, a stronger intermediate node, recursive chains, and cycles. Extend the CLI fixture/integration tests
in `tests/cli.rs` (and fixture Markdown where useful) to verify:

- a Next parent promotes only Ready dependencies to Next;
- an In-Progress parent promotes Ready and Next dependencies to In Progress;
- a stronger target is preserved and passes its stronger status to descendants;
- done/cancelled/custom targets remain unchanged;
- dry-run writes nothing, apply output and JSON distinguish Next versus In-Progress promotions, a second run is
  idempotent, and removing the active Pomodoro path still preserves existing unreachable-Next clearing semantics.

Finish with `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test` in `bob-cli`.

## Acceptance Criteria

- Pressing `!` on an eligible dependency bullet immediately raises a Ready/Next target to the valid parent task's higher
  Next/In-Progress status in same-note, cross-note, and counted workflows.
- Neither `!` removal nor either implementation ever lowers a dependency target because of status matching.
- `bob mark-next-tasks` produces the same status outcome for manually authored sole transclusions, carries the strongest
  state through recursive graphs, and remains cycle-safe and idempotent.
- Existing dependency metadata synchronization, Pomodoro structural cleanup, clearing of unreachable Next tasks,
  exclusions, warnings, dry-run safety, and terminal/custom status preservation continue to work.
- Human and JSON output truthfully report In-Progress promotions, all tests and validation commands pass, and the
  updated navigation plugin is synced from `bob-plugins` into the vault.
