---
create_time: 2026-07-11 17:34:24
status: done
tier: tale
---
# Plan: Retire Transclusions When Ctrl+Enter Closes Tasks

## Goal

Make the Obsidian `<Ctrl+Enter>` workflow retire every eligible embedded block-link reference to each task that the
keypress actually closes, including tasks reached during recursive Pomodoro transclusion traversal. A retired reference
must be a non-embedded, struck-through block link—for example, `![[Projects/Alpha#^review|Review]]` becomes
`~~[[Projects/Alpha#^review|Review]]~~`. Update `bob mark-next-tasks` to enforce the same canonical representation for
completed-task references beneath Pomodoros instead of recreating embeds, while preserving Next/dependency traversal,
Pomodoro history, dry-run guarantees, and idempotence.

## Current behavior and design constraints

- `task-status-cycler` routes `<Ctrl+Enter>` through several paths: a direct local checkbox toggle, full Pomodoro
  completion, direct completion of an embedded Pomodoro child, a root-only start for a bare Pomodoro child, and a
  generic embedded-task toggle. Full and direct embedded Pomodoro completion recursively traverse child `![[...#^id]]`
  links and may close multiple source tasks across multiple notes.
- Recursive completion currently reports only aggregate `visited`/`changed` state. It must retain the identities of
  tasks that changed to Done so cleanup can include descendants without treating already-Done traversed nodes as newly
  closed.
- Full Pomodoro completion intentionally excludes embedded bullets from the bullets copied into the next Pomodoro.
  Therefore reference retirement must happen only after the Pomodoro completion plan is built and applied; retiring
  links earlier would make them look like bare links and incorrectly carry completed work forward.
- Dependency navigation bullets remain represented by `[dependsOn::]` on the parent even after the dependency task is
  complete. The navigation plugin's reconciler currently recognizes only live transclusions and would recreate a retired
  embed unless the struck form becomes a managed terminal representation.
- `bob mark-next-tasks` currently does the opposite of the requested behavior: it embeds completed links and may move
  their bullet from an open/future Pomodoro to the current timed Pomodoro or last completed fallback. Its Pomodoro scan
  only records child links beneath open entries, so it cannot repair a stale embed already left under a completed
  Pomodoro.

## Behavioral contract

### `<Ctrl+Enter>` task closure

1. A task participates when that particular `<Ctrl+Enter>` action changes it from Todo (`[ ]`), Next (`[*]`), or In
   Progress (`[/]`) to Done (`[x]`). This includes direct local tasks, generic embedded-task toggles, the selected root
   and recursively reached descendants of an embedded Pomodoro child, and every recursively reached task closed while
   completing a Pomodoro. Reopening a Done task, starting a bare Pomodoro link, visiting an already-Done node, and
   skipped/canceled/custom statuses do not trigger retirement for that target.
2. Identify a closed target by its resolved Markdown path plus trailing block ID. A closed checkbox with no valid block
   ID still closes normally but has no link identity to retire.
3. After all status writes for the action succeed as far as they can, search Markdown notes for literal embedded block
   links that resolve to any newly closed `(path, block ID)` identity. Match same-note `![[#^id]]`, explicit paths,
   unique Obsidian linkpaths/basenames, aliases, and multiple occurrences by resolved identity rather than raw link
   spelling.
4. Rewrite only occurrences on descendant list bullets whose ancestry contains either a proper `#task` checkbox or a
   top-level checkbox in a `## Pomodoros` section. Ignore prose, unrelated list trees, fenced code examples, malformed
   links, heading/note links, and nonmatching block IDs. Preserve the bullet marker, indentation, aliases, neighboring
   text, nested descendants, and LF/CRLF style.
5. Canonicalize each matching token to `~~[[target#^id|alias]]~~`: remove the `!` immediately associated with the link
   and surround the complete wikilink token with Markdown's double-tilde strike markers. If an unusual stale form is
   already struck and embedded, remove only the embed marker rather than double-wrapping it. Repeated cleanup is a
   no-op.
6. Treat cleanup as the final phase of the keypress. Recursive graph discovery must finish before links are changed, and
   full Pomodoro completion must apply its local completion/carry-forward plan before retirement. Batch and dedupe
   closed identities so cycles, repeated links, and multiple roots cause one coordinated scan rather than competing
   writes.
7. Keep task closure best-effort as today: an unreadable or concurrently changed reference note must not roll back
   successful status changes, but failures should be logged and summarized in one notice rather than silently causing
   unrelated edits. Use the live active-editor buffer for the active note and guarded vault processing for other notes
   so unsaved editor content is not overwritten.

### Dependency navigation compatibility

- Teach `bob-navigation-hotkeys` to parse `~~[[...#^id]]~~` as the managed terminal form of a dependency navigation
  bullet when its block/task identity belongs to the parent's `[dependsOn::]` set.
- Dependency reconciliation must preserve that struck form while the dependency remains listed, rather than replacing it
  with `![[...]]` or inserting a duplicate embed. Adding/removing other dependencies, legacy consolidation, cross-note
  path preservation, and migration transforms must remain idempotent; removing the dependency property may remove its
  managed struck bullet through the existing removal semantics.
- Plain arbitrary struck links and `#^ref` links remain unmanaged and protected, just like unrelated child content
  today.

### `bob mark-next-tasks`

1. Separate the Pomodoro model's two concerns:
   - only block links beneath open Pomodoros feed the direct desired-task set and the existing recursive dependency
     closure;
   - block-link bullets beneath both open and completed Pomodoros are candidates for completed-reference repair.
2. Resolve task completion exactly as today: conventional `[x]`/`[X]` plus configured Obsidian Tasks `DONE` symbols,
   with ambiguous, unresolved, conflicting, canceled, and non-task targets warned and left untouched.
3. Canonicalize every unambiguously completed Pomodoro block-link occurrence—whether currently plain, embedded, or
   already struck—to `~~[[...]]~~`. Preserve aliases and mixed bullet text, change only completed link tokens, remove
   any `!`, avoid duplicate strike markers, and make a second run a no-op.
4. Preserve the existing relocation policy for completed links discovered under open/future Pomodoros: move the whole
   bullet subtree to the current timed Pomodoro, otherwise the last completed fallback, otherwise normalize in place. A
   repair found under an already-completed Pomodoro is normalized in place and is never moved into a newer session; this
   protects historical Pomodoro grouping.
5. Struck dependency bullets are no longer live dependency edges because dependency traversal continues to require a
   sole `![[...#^id]]` child bullet. Completed dependency tasks remain terminal and are not promoted; live transcluded
   dependencies continue to recurse and handle cycles exactly as before.
6. Replace human wording such as “embed completed references” with “retire”/“strike completed references,” and count
   each normalized link once even when both `!` removal and strike insertion occur. Add a `struck_completed_references`
   JSON list containing the link identity, Pomodoro context, and whether an embed marker was removed. Retain
   `embedded_completed_references` as a deprecated, always-empty compatibility field for one contract cycle; keep
   `moved_completed_references` and all Next/dependency fields stable. Update the no-op decision and summary counts
   accordingly.
7. Keep the existing preflight, all-output composition, atomic write, `--dry-run`, warning, and multiple-current
   Pomodoro guard behavior. Daily-note structural edits must still compose safely with status edits when the daily note
   itself contains Tasks tasks.

## Implementation approach

1. **Add pure link-retirement parsing and rewriting to `task-status-cycler`.** Build reusable helpers that find fenced
   ranges and list ancestry, parse embedded block-link token spans including aliases/strike envelopes, resolve each
   candidate from its origin note, and produce stable per-file edits for a set of closed identities. Keep qualification
   independent from the recursive completion code so it can be unit-tested against realistic Markdown.

2. **Return concrete closure results from every `<Ctrl+Enter>` close path.** Extend direct and resolved status writes to
   report the target identity and whether the transition actually reached Done. Thread a shared collector through
   recursive completion, while retaining the current visited/depth/target guards and best-effort sibling behavior. The
   bare-link start path returns no closures; toggling Done back open returns no closure.

3. **Coordinate cleanup after the action.** Introduce one post-close coordinator used by direct local completion,
   generic embedded completion, direct Pomodoro-child recursion, and full Pomodoro completion. For full Pomodoros,
   invoke it only after the local completion plan has prevented embedded bullets from being copied forward. Serialize
   overlapping cleanup batches, apply active-note edits through the editor, use vault `process`/revalidation elsewhere,
   and aggregate failures into one diagnostic.

4. **Preserve terminal dependency bullets in `bob-navigation-hotkeys`.** Extend the managed dependency bullet parser,
   collection model, formatter, and sync planner with a live-transcluded versus terminal-struck state. Reconciliation
   should retain the existing terminal state for matching desired dependencies while still emitting canonical live
   embeds for new dependencies and honoring current legacy/removal behavior.

5. **Invert and broaden the CLI's completed-reference structural plan.** Record exact link spans and formatting state,
   scan completed Pomodoro entries for repair without adding them to active reference counts, generate token
   canonicalization edits instead of embed offsets, and restrict moves to bullets sourced from open entries. Apply token
   edits and subtree moves from stable source spans before composing them with task-status replacements.

6. **Update the public contract.** Revise `bob mark-next-tasks` clap help, runner/README summaries, and
   `docs/mark-next-tasks.md` examples, transition language, JSON schema, warnings, relocation rules, and idempotence
   notes. Update plugin README descriptions and bump the manifests for every changed plugin.

## Tests and verification

### Plugin tests

- Add pure retirement tests for same-note and cross-note links, explicit paths and aliases, multiple matching links,
  mixed matching/nonmatching links, existing strike wrappers, tabs/spaces, LF/CRLF, fenced examples, malformed links,
  non-task list trees, nested task descendants, Pomodoro descendants, unresolved links, and idempotence.
- Add runtime tests proving direct Todo/Next/In-Progress closure retires references, Done-to-open does not, a generic
  embedded toggle reports its resolved closure, and a strict bare Pomodoro link only starts its target.
- Expand recursive tests so newly closed roots and descendants are all collected, already-Done traversed parents are not
  reported, cycles/duplicate roots terminate, and references in both active-editor and vault-backed notes are rewritten.
- Add a full Pomodoro regression test showing embedded completed bullets remain under the completed source Pomodoro,
  become struck only after completion planning, and are not copied into the newly created placeholder; preserve plain
  unfinished links and note bullets as today.
- Add navigation tests for parsing and collecting managed struck dependency bullets, preserving them while adding or
  removing sibling dependencies, avoiding duplicate live embeds, protecting unrelated struck/ref links, cross-note
  formatting, migration idempotence, and ordinary live `!` toggle behavior.
- Run `npm test` and `npm run validate`, then deploy with `bob plugins sync -r <bob-plugins-repo>` and perform a scratch
  vault/manual Obsidian check for direct, recursive, and full-Pomodoro `<Ctrl+Enter>` flows.

### CLI tests

- Add focused Rust tests for parsing plain/embedded/struck link spans; canonical token rewriting with aliases and mixed
  content; completed-entry in-place repair; open-entry relocation; historical-entry non-relocation; custom DONE and
  conflicting duplicate statuses; fenced and unresolved protection; LF/CRLF preservation; structural/status edit
  composition; and second-run idempotence.
- Keep dependency tests proving only live sole transclusions form edges; add a struck completed dependency case proving
  it is terminal and does not get re-transcluded or promoted.
- Update `tests/fixtures/mark_next/` and CLI integration assertions so dry-run writes nothing, JSON reports struck and
  moved references with the compatibility field intact, human output says retired/struck rather than embedded, apply
  yields `~~[[...]]~~`, completed historical Pomodoros are repaired in place, and reruns report already in sync.
- Run targeted `mark_next`/Pomodoro tests followed by the repository's full `just` formatting, lint, unit, CLI, and
  parity pipeline.
- Finish with `bob mark-next-tasks --dry-run --format json` against the live vault and inspect the proposed
  retired/moved references before any real vault write. After explicit diff review, apply once and confirm the next dry
  run is a no-op.

## Expected files

### `bob-plugins`

- `plugins/task-status-cycler/main.js`
- `plugins/task-status-cycler/manifest.json`
- `plugins/bob-navigation-hotkeys/main.js`
- `plugins/bob-navigation-hotkeys/manifest.json`
- `scripts/test-task-status-cycler.cjs`
- `scripts/test-navigation-hotkeys.cjs`
- `README.md`

### `bob-cli`

- `src/native/mark_next.rs`
- `src/runner.rs`
- `tests/cli.rs`
- `tests/fixtures/mark_next/...`
- `docs/mark-next-tasks.md`
- `README.md`

## Sequencing and risk controls

1. Implement/test pure retirement and managed-terminal dependency helpers.
2. Thread closed identities through all `<Ctrl+Enter>` paths and add the delayed, batched cleanup coordinator.
3. Invert/broaden CLI normalization and update output/docs/fixtures.
4. Run both repositories' complete checks, deploy plugins, and manually verify sequencing in a scratch note.
5. Run a live-vault CLI dry run, review the exact diff, then apply only after that review and verify idempotence.

The highest risks are rewriting unrelated embeds, racing active-editor content, copying a newly bare completed link into
the next Pomodoro, recreating terminal dependency embeds, and moving historical completed-session bullets. Resolved
path-plus-block matching, ancestry/fence guards, delayed cleanup, guarded per-file processing, terminal-state-aware
dependency reconciliation, and separate active-versus-historical CLI scans address those risks directly.
