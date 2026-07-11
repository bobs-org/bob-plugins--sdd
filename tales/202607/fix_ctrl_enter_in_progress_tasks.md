---
create_time: 2026-07-11 12:46:51
status: wip
prompt: .sase/sdd/prompts/202607/fix_ctrl_enter_in_progress_tasks.md
---

# Fix Ctrl+Enter for in-progress Obsidian tasks

## Goal

Make the Task Status Cycler plugin's `<Ctrl+Enter>` open/done action complete an in-progress `[/]` task, while
preserving the established behavior for Todo `[ ]`, Next `[*]`, Done `[x]`, canceled `[-]`, Tasks-managed metadata,
transcluded task links, and Pomodoro-specific completion flows.

## Findings

- The Vim mappings for both `<C-CR>` and `<C-Enter>` are already registered to `taskStatusCyclerToggleTaskOpenDone`, so
  the keymap registration itself is working.
- The action parses the active Markdown task and takes the direct toggle path only when `isOpenDoneTaskStatus()` accepts
  its checkbox symbol.
- That policy currently accepts Todo, Next, and Done but explicitly rejects In Progress. An ordinary `[/]` line
  therefore falls through to the Pomodoro/transclusion handlers, none of which handles a regular selected task,
  producing the observed no-op.
- The existing status setter already has the desired write semantics: an eligible incomplete `#task` is sent through the
  Obsidian Tasks `set-status-symbol-to-x` command, with a local metadata-aware fallback, while a non-Tasks checkbox uses
  a local symbol replacement.
- Recursive Pomodoro completion already has a separate, explicit policy that includes In Progress, so this fix does not
  need to change that workflow.

## Implementation plan

1. In the `bob-plugins` linked repository, update `plugins/task-status-cycler/main.js` so the shared open/done
   eligibility policy includes In Progress (`/`). Keep the transition rule as “Done reopens to Todo; every eligible
   incomplete state completes to Done,” and continue excluding canceled or unknown custom statuses. This lets the
   existing Vim action, editor command, Tasks-command dispatch, metadata-aware fallback, and single-target transclusion
   toggle all use the same consistent behavior without adding a key-specific special case.

2. Extend `scripts/test-task-status-cycler.cjs` with focused regression coverage:
   - Update the direct transition matrix to require `[/] -> [x]`, while retaining assertions for Todo, Next, Done, and
     canceled behavior.
   - Exercise the registered `<Ctrl+Enter>` Vim action on an in-progress `#task` and assert that it dispatches the Tasks
     plugin's Done command. Preserve coverage that both CodeMirror key spellings map to the same normal-mode action and
     that the existing Next behavior remains supported.
   - Keep the recursive-completion tests unchanged except for any wording needed to clearly distinguish the direct
     toggle policy from its already-correct Pomodoro policy.

3. In `bob-plugins`, run the complete plugin test suite (`npm test`) and manifest/source validation (`npm run validate`)
   to catch behavioral, syntax, and plugin-layout regressions.

4. Deploy the corrected source-of-truth plugin to the Obsidian vault with `bob plugins sync -p task-status-cycler`, as
   required by the linked repository workflow, and verify the plugin reports synchronized (or equivalently that a
   subsequent dry run has no pending copy).

## Expected result

With the cursor on `- [/] #task ...`, pressing `<Ctrl+Enter>` changes the task to Done through the same Tasks-aware path
used by Todo and Next tasks. Pressing it on Done still reopens the task as Todo; canceled and unsupported custom
statuses remain untouched; Pomodoro recursive completion continues to behave as before.
