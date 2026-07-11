---
create_time: 2026-07-11 18:53:07
status: done
prompt: .sase/sdd/plans/202607/prompts/fix_review_findings_20260711.md
tier: tale
---

# Fix verified bugs from review of 2026-07-10..11 commits (bob-cli + bob-plugins)

## Background

All commits landed in the last two days (2026-07-10 through 2026-07-11) in bob-cli (~23 commits: the mark-next-tasks
engine, the Tasks v8 native query engine `bob-cli-9.1..9.8`, capture/pomodoro routing, projects scheduling, collect-done
dependency-identity repair) and in the linked `bob-plugins` repo (12 commits: bob-navigation-hotkeys,
task-status-cycler, block-id-prompt, migration scripts) were reviewed by eight parallel review agents. Findings below
marked **[repro]** were reproduced against the built binary or via extracted plugin helpers; the rest were traced in
code. All existing test suites pass at HEAD in both repos, so every fix needs a regression test that would have caught
it.

Line numbers refer to current HEAD of each repo (bob-cli `5a22c43`, bob-plugins `b59374e`).

This plan fixes the verified bugs plus low-risk objective improvements, organized into six largely independent
workstreams (A–D bob-cli, E–F bob-plugins). A final section lists findings deliberately deferred (feature-sized or
ambiguous intent).

To touch bob-plugins, implementing agents must open the linked repo via
`sase workspace open -p bob-plugins -r "<reason>" <workspace_num>` and work only in the printed path.

## Cross-cutting design note: dependency-ID grammar

The new path-qualified dependency identity (`path__with__underscores__blockid`) only supports vault paths matching
`[A-Za-z0-9_/-]`. Three independent findings (A2, E1, F2 below) show code _throwing or aborting whole runs_ when a path
can't be encoded (spaces, dots, unicode — spaced names already exist in the live vault). The objective fix now is
**graceful degradation everywhere**: identity generation failure must never abort a run, throw uncaught, or spam notices
— skip that note's identities, surface a single clear warning, and continue. Extending the encoding to support arbitrary
paths is a design decision for Bryan, deferred (see final section).

---

## Workstream A — bob-cli: collect_done.rs (dependency identity repair, commit 0e67766)

**A1. Block-id lexing must match Obsidian's grammar (end-of-line anchors only).** [repro] `block_id_occurrences_in_text`
(`src/native/collect_done.rs:2214-2254`) matches any whitespace-preceded `^token` mid-line, so prose like
`10 ^2 equals 100` mints identities. Obsidian block anchors are end-of-line only; the tasks parser already uses
` \^([a-zA-Z0-9-]+)$` (`src/native/dataview/tasks/task.rs:10`). Fix the lexer to end-of-line anchors and align
`is_block_id_byte` (`:2256`) to `[A-Za-z0-9-]` (drop `_`). This removes the false-positive collision rejections (repro:
`a/b.md` prose `see ^c for details.` colliding with `a__b.md#^c`) and shrinks A2's blast radius.

**A2. Unqualifiable note paths must not abort the run.** [repro] `dependency_id` (`:2128-2148`) errors on any path
outside `[A-Za-z0-9_/-]`, and the `?` chain through `validate_dependency_identity_index` (`:1199`),
`archived_block_targets` (`:1491`), and `overlay_current_run_block_targets` (`:1539`) aborts the entire
`bob move-done-tasks` / `bob nightly` archive step (repro: vault with `Untitled 1.md` containing `10 ^2`). Fix: skip
identity generation for unqualifiable paths during scanning/indexing (collect a per-run warning listing skipped notes);
only error if an actual move/repair requires an identity that cannot be generated.

**A3. Dependency-metadata repair must skip fenced code and inline code spans.** [repro] `repair_dependency_metadata`
(`:1673-1749`) rewrites `[id::]`/`[dependsOn::]` tokens inside ` ``` ` blocks and backtick spans (repro: docs examples
rewritten to archive-qualified ids). Reuse/introduce a shared fence-scanner (see C6) and skip fenced lines and
inline-code spans.

**A4. Repair regex misses field forms the Tasks engine accepts.** The regex at `:1694`
(`\[(id|dependsOn)::([^\]\n]*)\]`) misses parenthesized `(dependsOn:: …)` and bracket-with-leading-space
`[ dependsOn:: …]`, both accepted as real dependency fields by `bob query --tasks` — such dependents silently keep stale
ids after a move. Extend the repair matcher to the same field grammar the tasks parser accepts.

**A5. Improvements:** hoist the `:1694` regex into a `static LazyLock` (currently compiled per note per run); share one
read pass between `validate_dependency_identity_index` (`:1197`) and `build_collection_plan` (currently every non-done
note is read twice, three times with `:1341`); make the test helper `moved_targets` (`:4116-4133`) call the production
`dependency_id` instead of duplicating it.

## Workstream B — bob-cli: capture.rs + pomodoro.rs (commits 15ec5ac, f4a60e7)

**B1. Time-of-day text is consumed as a Pomodoro marker.** [repro] `is_pomodoro_marker_candidate`
(`src/native/capture.rs:1025-1038`) puts no shape constraint on route/id, so `bob capture "call dentist @5:30pm"` routes
to note `5.md` with block `^30pm` (silent misroute, exit 0). Fix: a terminal `@route:id` token is a marker candidate
only when the route segment contains at least one ASCII letter (routes name notes; `5`, `10`, `12` from times never
qualify). Legacy `@route!id` unaffected. Add regression tests for `@5:30pm`, `@10:00`, and a valid `@dev:foo`.

**B2. Leading route must win over trailing malformed marker candidates.** [repro] `validate_special_terminal_markers`
(`:1040-1049`) runs before the leading-route branch, so `bob capture "@groceries ping @x:"` exits 2 instead of routing
to groceries with body `ping @x:`, contradicting the comment contract at `:910-911`. Fix the ordering: when a leading
route is present, later route-looking tokens are body text.

**B3. Ledger scanning must be fence-aware.** [repro] The open-entry selection loop in `insert_pomodoro_block_link`
(`:547-561`) and `latest_ledger_pomodoro` (`src/native/pomodoro.rs:249-264`) treat entries inside ` ``` ` blocks in the
Pomodoros section as real (repro: block link inserted inside a code fence). Track fence state in both loops, mirroring
`pomodoros_section_range`.

**B4. Open-entry semantics.** [repro] `open_ledger_task` (`src/native/pomodoro.rs:365-381`) counts every checkbox except
`x`/`X` — and even plain non-checkbox bullets — as open. A canceled `- [-]` timed entry makes every capture that day
fail with "multiple open timed Pomodoros"; a plain note bullet can be selected as the link anchor. Fix: canceled (`-`)
counts as closed; only checkbox bullets are candidates for anchor selection.

**B5. Lone `---` at the top of a note must not swallow the file.** [repro] `pomodoros_section_range`
(`src/native/pomodoro.rs:272-282`) treats an unclosed leading `---` as frontmatter forever, hiding the whole note from
status/capture. Only treat the block as frontmatter when a closing delimiter exists; otherwise it's content (matches
Obsidian).

**B6. Improvements:** update the stale top-level help example `bob capture '@!dev:foobar'` (`src/runner.rs:255`) to the
canonical colon form; add a duplicate-link check on the daily ledger before inserting `[[route#^id]]`
(`src/native/capture.rs:504-505`) so re-captures after the target check passes don't append identical links.

## Workstream C — bob-cli: mark_next.rs (commits bc829fa, 1ca1109, 2677dea, 5a22c43)

**C1. Dotted note names resolve to the wrong note.** [repro] `target_to_markdown_path`
(`src/native/mark_next.rs:1299-1315`) uses `set_extension("md")`, so target `Notes.v2` becomes `Notes.md` — the wrong
note's task gets promoted, silently. Fix: append `.md` only when the target doesn't already end with it; the basename
fallback in `NoteIndex::resolve` (`:1274-1296`) must compare the full target name, not `file_stem`.

**C2. Retired (struck) references must not keep promoting dependency chains.** [repro] `scan_pomodoros` (`:625-630`)
feeds canonical `~~[[...]]~~` links into `raw_references`, so a retired completed task's dependency chain is re-promoted
to Next forever (contradicts the retirement contract in docs/mark-next-tasks.md). Fix: struck canonical links are
excluded from the desired-promotion set and from dependency-closure roots; they remain visible only in retirement
reporting.

**C3. Fence check ordering truncates dependency scanning.** [repro] In `dependency_edges` (`:1350-1364`) the
`indentation <= source_indent` break runs before the `fenced.contains` check, so column-0 content inside an indented
fence ends the subtree scan and drops later dependency bullets. Check fence membership first.

**C4. Relocation to the completed-fallback entry must not demote live references.** [repro] `plan_structural_changes`
(`:901-994`) moves a mixed bullet (completed + live links) under the last completed Pomodoro when no open one exists;
the desired-set recomputation then clears the live link's Next status. Fix: when the relocation target is a closed
entry, don't relocate bullets that still carry live references (retire/strike the completed links in place instead).

**C5. Strikethrough detection must be span-aware.** [repro] `block_link_occurrences` (`:706-716`) inspects only two
characters each side, so (a) a link between two separate `~~…~~` spans is misclassified as already-canonical and never
retired, and (b) striking a link whose prefix ends with `~~` emits a 4-tilde run that GFM won't render. Parse the line's
`~~` spans (pair delimiters left-to-right) to classify links, and when wrapping adjacent to an existing span, emit a
separating space so the new span renders.

**C6. Improvements:** drop the unused `.clone()`s at `:493/:502`; remove the no-op `#[serde(default)]` at `:198`; remove
the redundant unchanged-check in `compose_outputs` (`:1546-1560`); early-continue in `dependency_edges` for files with
no block-id tasks (`:1341`); canonicalize `daily_path` once instead of per-file (`:357`, `:1537`); extract the
duplicated `MarkdownFence` helpers (`:778-824` vs `src/native/pomodoro.rs:307-339`) into one shared module (also reused
by A3/B3).

## Workstream D — bob-cli: projects.rs (commits 4c51d3b, ddb1e54, e7b1e3a)

**D1. Overlapping `#hide` removal edits corrupt notes.** [repro — data corruption] `task_visibility_edits`
(`src/native/projects.rs:1510-1534`) emits per-tag removal ranges that overlap across shared whitespace
(`inline_field_removal_range`, `:1839-1859`); `apply_project_changes` (`:1455-1466`) then applies them against shifted
positions, deleting newlines and merging task lines (repro: `- [ ] #task Work #hide #hide` + next line merged into one).
Fix: compute one merged removal range per line (single-pass rebuild of the task text); additionally make
`apply_project_changes` reject overlapping `TextEdit`s with a per-file error as defense-in-depth.

**D2. An invalid `scheduled:` on a child must not cascade into parent edits.** [repro] A child excluded from the
children map for a schedule-only issue (`:1021-1034`, `:951-954`, `:731-735`) causes sync to strip `[[Child]]` from the
parent ledger and un-hide the parent's `^prj` (`:1263-1269`, `:1216-1219`), contradicting docs/projects.md:49-50. Fix:
schedule-parse issues must not remove a file from sub-project consideration (`SubprojectState` doesn't depend on
`scheduled`); only skip edits to the invalid file itself.

**D3. Frontmatter keys must match at column 0.** [repro] `parse_project_schedule` (`:2026-2030`) matches `scheduled:`
inside indented block-scalar lines (`notes: |`), producing spurious per-file errors, exit 1, and the D2 cascade. Require
top-level keys unindented; harden `frontmatter_value` (`:2119-2127`) and `frontmatter_line_has_key` (`:1890-1894`) the
same way.

**D4. Terminal projects must not get schedule visibility edits.** [repro] The scheduled block in `plan_project_sync_at`
(`:1182-1205`) sits outside the `!effective_status.is_terminal()` gate (`:1207`), so a canceled project with a past
`scheduled:` gets `#hide` _removed_ — dead tasks surface on the dash — and done projects get `^prj`-line edits,
contradicting docs/projects.md:128-131. Fix: gate the scheduled reconcile on non-terminal status; update the
`FutureClosed` assertion in tests/cli.rs (~:4370) and docs accordingly.

## Workstream E — bob-cli: Tasks query engine (commits ef58bb8..6647342, bob-cli-9.1..9.8)

Parser / index / task model:

**E1. Boolean combinations: support 3+ operand chains and quotes inside operands.** [repro] `parse_boolean`
(`src/native/dataview/tasks/parse.rs:1184-1234`) splits at the first top-level operator and requires both sides to be
single wrapped filters, so `(done) OR (is blocked) OR (has due date)` fails;
`find_top_level_operator`/`strip_wrapping_delimiters` (`:1247-1332`) treat any `'`/`"` on the line as quote state, so
`(description includes John's) AND (not done)` fails. Fix: tokenize top-level operands and operators, build the AST
left-associatively with Tasks precedence (NOT > AND > XOR > OR), and recognize delimiters only at operand boundaries
(apostrophes/quotes inside operands are literal, matching Tasks 7+).

**E2. Date filters: numbered ranges and weekday semantics.** [repro] `looks_like_date_expression` (`parse.rs:1569-1621`)
and `parse_date_range` (`src/native/dataview/tasks/filter.rs:397-446`) reject `YYYY`, `YYYY-MM`, `YYYY-Qn`, `YYYY-Www` —
all valid Tasks v8 range filters — and resolve bare weekdays only forward, while Tasks resolves to the _closest_
occurrence (docs: on Wednesday, `tuesday` means yesterday). Implement numbered-range parsing with correct
on/before/after/in range semantics, closest-occurrence bare weekdays, and `next/last <weekday>`. (Free-text dates like
`25th May 2023` / `in two weeks` are deferred.)

**E3. `priority above/below/not X` without `is`.** [repro] `parse.rs:1439-1471` requires the literal `is`; Tasks makes
it optional. Accept both.

**E4. Frontmatter handling in the index.** [repro] `index.rs:130-142`: an unclosed leading `---` drops every task in the
file; an indented `---` opens frontmatter; `...` closes it. Obsidian accepts none of these — frontmatter is only a
strictly-closed block opened at line 0 column 0. Fix and unify with the second grammar in `parse.rs:491-508`.

**E5. Structure parsing details.** [repro for the first two] (a) `index.rs:177-196`: a column-0 paragraph between lists
doesn't clear the list `stack`, so a later indented task is wrongly attached as a child of the previous list (affects
tree mode, `is_root`, sub-item filters) — clear the stack on non-list content lines. (b) `index.rs:334` and the copy in
`mod.rs:395-399` strip trailing `#` unconditionally, so `## Learning C#` becomes `Learning C` — CommonMark only strips a
closing sequence preceded by whitespace. (c) `task.rs:461-469` accepts `1)`-style list markers Tasks ignores — restrict
to `[-*+]` and `\d+\.`. (d) `index.rs:338-344` setext underlines require ≥3 chars; CommonMark accepts 1+. (e)
`parse.rs:1411-1414` `status.type` prefix match needs a word boundary (`status.typeis TODO` currently parses).

**E6. Invalid due date urgency.** `calculate_urgency` (`task.rs:866-879`) skips invalid dues; Tasks scores them like
far-future dates (dueScore 0.2 → +2.4). Match that.

Filters / JS / result / render / mod:

**E7. Tree mode renders a matched task twice.** [repro] `render_group_tasks`
(`src/native/dataview/tasks/render.rs:51-98`) skips a task only when its _immediate_ parent is in the group, while
`render_tree` walks all descendants — a matched grandchild under a filtered-out parent appears both nested and at top
level. Skip when _any_ ancestor is in the group.

**E8. `group by function` null/empty semantics.** [repro] `js.rs:167-172` + `result.rs:277-305` silently drop tasks
whose grouping function returns `null`/`[]` (count changes); Tasks docs: `null`, `''`, `[]` all mean "no heading, task
still displayed". Also an empty-string group renders a bare `#### ` heading (`render.rs:22-29`) — render no heading
instead.

**E9. Per-block error isolation in `run_note`.** [repro] `mod.rs:88-95` aborts the whole note on the first invalid
block; Obsidian renders each block independently with the error inline. Execute every block, attach the error to that
block's human/JSON output, and exit non-zero only if at least one block failed.

**E10. ```tasks blocks inside callouts/blockquotes.** [repro] `opening_fence` (`mod.rs:356-368`) never matches
`> `-prefixed fences, so queries in callouts are invisible to `--tasks-note`. Strip blockquote prefixes when extracting
blocks.

**E11. String collation.** [repro] `natural_compare` (`result.rs:582-615`) uses raw codepoint order ("Banana" <
"apple"); Tasks uses `localeCompare(…, {numeric: true})`. Implement case-insensitive, numeric-aware natural comparison
for sorts and group-name ordering (full ICU collation out of scope).

**E12. Smaller parity fixes.** [all repro] (a) `filter.rs:280-285` global-filter removal removes all occurrences; Tasks
removes only the first. (b) `js.rs:98-103` `task.priority` must be the Tasks enum string digits `'0'..'5'`, not
`"Normal"`. (c) `result.rs:429-439` preset-sourced statements are omitted from `explain` output — include them. (d)
`render.rs:106-113` `hide tags` strips any `#`-prefixed word and collapses whitespace — remove only tags recognized in
`task.tags`.

**E13. Sort-by-function robustness + perf.** `compare_function_sort` (`js.rs:407-433`) re-evaluates the JS key per
comparison, and after a JS error the comparator returns Equal for remaining pairs (inconsistent order can panic modern
Rust sorts, `result.rs:126-170`). Precompute keys per task before sorting; a key-evaluation error fails the query
cleanly.

**E14. Improvements:** compile filter regexes once per filter instead of per task (`filter.rs:240-247`); reuse one
sandbox/index across blocks in `execute` (`mod.rs:98-134`); pin `moment.utc()`/`moment.now` to `BOB_NOW` like bare
`moment()` (`js.rs:34-47`); index groups by name path (`result.rs:290-305`); drop the stray blank line when explanation
is empty (`render.rs:16-19`); hoist the placeholder regex to `LazyLock` (`parse.rs:700`); make `apply_dependencies`
linear via id maps (`index.rs:299-319`); use `0..20` extraction runs for Tasks parity (`task.rs:538`); unknown
`taskFormat` in data.json falls back to default instead of failing all queries (`settings.rs:53-62`); preserve original
spacing in `remove_global_filter_as_word` (`task.rs:708-717`).

## Workstream F — bob-plugins: bob-navigation-hotkeys

**F1. `dependencyId()` must never throw uncaught through UI flows.** [repro] The throw at
`plugins/bob-navigation-hotkeys/main.js:2185-2200` propagates uncaught from the dependency picker and related flows
(call sites at `:792`, `:821`, `:7365-7373`, `:8180`, `:8395`, `:8527`) whenever the note path has a space/dot/unicode
char — the modal is left half-switched with no Notice. Fix per the cross-cutting note: guard all call sites;
unqualifiable candidate tasks are skipped (or the feature declines with one clear Notice); the picker keeps working.

**F2. Navigation bullets must be keyed by full identity, not bare block ID.** [repro — data loss] Dedupe/collection by
bare id (`collectDependencyNavigationBullets` `:1243-1249`, `normalizeDependencyNavigationTargets` `:886-893`, same-file
resolution at `:1222-1227`) makes `![[#^x]]` and `![[Other#^x]]` collide: consolidation and even unrelated picker syncs
delete the cross-note bullet (`transformDependencyBulletsInContent` `:5356-5427`, plan delete `:1431-1442`), and
cross-note bullets aren't collected at all without `additionalManagedIds`. Key collection, dedupe, normalization, and
sync plans by qualified identity (note + block id) so distinct-note bullets never collide and cross-note bullets are
managed first-class.

**F3. Transclusion toggles must skip struck (retired) dependency lines.** [repro]
`findTransclusionToggleTargets`/`parseTransclusionToggleTargetAt` (`:4900-4988`) toggle `~~[[#^a]]~~` into
`~~![[#^a]]~~`, an unmanaged shape that then duplicates on next sync. Skip struck dependency bullets in single-line and
counted-range (`:9360-9410`) toggles.

**F4. Shrink the cross-file write window in `applyDependencyAwareTransclusionChanges`.** Target `[id::]` overwrite and
vault-wide propagation (`:9167-9263`, `:9266-9302`) complete before the final source-editor guard at `:9260`, so a user
edit mid-flight leaves a dangling legacy `dependsOn`. Re-verify the source editor content immediately before starting
external writes and abort cleanly if it changed (keep the final guard too).

**F5. Truthful UI text.** [repro] The block-ID prompt preview (`:8126-8140`) and comments (`:776-789`, `:8175-8179`,
`:8392-8394`) say the existing `[id:: …]` is kept; `applyPromptedBlockIdToTaskLine` (`:790-794`) overwrites it with the
canonical id. Fix the preview/comments to state the replacement.

**F6. `rewriteDependsOnIdsInLine` must accept space-variant fields.** [repro] `:1692-1693` hard-codes `\[dependsOn::`;
`parseBulletPropertyFields` (`:514-540`) accepts `[dependsOn :: …]`, so those lines are managed everywhere except
normalization. Align the regex.

**F7. Migration script ambiguity handling.** `scripts/migrate-dependency-bullets.mjs:79-117` resolves duplicate bare
block IDs first-file-wins and lets `[id::]` entries silently hijack other files' block ids. Emit per-ID ambiguity
warnings and skip rewriting ambiguous references (never guess on `--write`).

**F8. Dash-restore notice suppression.** `:11107-11150` shows "No active markdown editor" after the retry window even
when the user deliberately navigated to another file. Suppress the notice when the active file changed during the
retries.

**F9. Improvements:** prefilter `propagateDependencyIdReplacements` (`:9266-9302`) with `cachedRead` +
`includes(legacyId)` before `vault.process`, and continue past single-file failures; tolerate/normalize alias forms in
`DEPENDENCY_TRANSCLUSION_BULLET_RE` (`:213-214`) so an added `|alias` doesn't unmanage a bullet; register the
`markOpenTaskJumpDispatch` timeout (`:9985-10006`); precompute a per-content line-context array to de-quadratic
`isObsidianTaskAtLine` callers (`:6651-6660`).

## Workstream G — bob-plugins: task-status-cycler + block-id-prompt + migration

**G1. Rename must rewrite same-file `dependsOn` references.** [repro — broken dependency graph]
`rewriteRenamedDependencyIds` (`plugins/task-status-cycler/main.js:2022-2050`) rewrites only `[id::]` fields and
`propagateDependencyBlockIds` (`:2997-3000`) skips the origin file, so a renamed note's own `[dependsOn:: Old__a]` keeps
pointing at the old path and can never be repaired. Rewrite dependsOn values referencing the renamed note's old path
inside the renamed file too; add the missing regression test (`scripts/test-task-status-cycler.cjs` covers only
cross-file dependents).

**G2. `dependencyId` graceful degradation.** [repro] Same defect family as F1 (`:1843-1857`): spaced-path notes make the
runtime normalizer silently no-op via `.catch(() => {})` (`:2807`, `:2828`) and rename reconciliation abort with a
spurious error Notice (`:2742-2749`). Guard per the cross-cutting note: skip unqualifiable files with one informative
Notice; never throw, never spam.

**G3. Runtime normalizer must skip fenced code.** [repro] `normalizeTaskDependencyBlockIds` (`:1982-2020`) rewrites
`[id::]` examples inside
```fences and even appends`^blockid`there. Reuse`getFencedLineNumbers` (`:1629-1653`); unify it with the fence regexes at `:621-622`
into one scanner.

**G4. Identity migration script must skip fenced code.** `scripts/migrate-task-dependency-identities.mjs`
(`getTaskIndex` `:85-138`, dependsOn scan `:242-285`) indexes and rewrites fenced examples, and fenced content can
fatally fail the `--write` preflight (`:226-235`, `:362-365`). Add fence detection to both passes.

**G5. Editor normalizer stale-snapshot guard.** `normalizeActiveEditorDependencyBlockIds` (`:2838-2891`) and
`reconcileRenamedDependencyIds` (`:3090-3121`) await a full-vault scan, then `replaceRange` against stale coordinates —
typing during the scan gets clobbered. Mirror the vault path's `text === snapshot` guard: re-check the editor content
before applying edits, bail (and reschedule) if it changed.

**G6. Non-Vim toggle must retire references.** [verified: 3 of 4 close paths retire] `handleToggleOpenDoneCommand`
(`:4570-4586`) closes tasks without calling `retireClosedTaskReferences`, unlike the Vim path (`:3477-3490`). Wire
retirement into the command path.

**G7. Struck links are not pending work.** [repro] `classifyPomodoroSubBullets` (`:1343-1359`) matches `~~[[a#^x]]~~` as
a copyable task link, so retired references are carried into every new Pomodoro forever (`buildPomodoroCompletionPlan`
`:1481-1505`). Classify struck links as non-copyable.

**G8. Retirement ancestry must stop at paragraph boundaries.** [repro] `hasEligibleRetirementAncestor` (`:1667-1704`)
continues past non-list lines, striking indented embeds that belong to later prose, not the task. Terminate the ancestor
scan at non-list content lines.

**G9. `alreadyStruck` must be span-aware.** [repro] `:1761-1769` treats `~~before~~![[A#^review]]~~after~~` as already
struck and rewrites it to an _unstruck_ plain link that can never be retired. Use the same `~~`-span pairing approach as
C5.

**G10. Ambiguity path: stop rescanning and re-notifying every edit.** While any ambiguous generated id exists, every
debounced edit triggers a full-vault read plus a duplicate Notice (`:2843-2857`, `:2924-2936`,
`findAmbiguousDependencyIds` `:2945-2976`). Cache the scan result per debounce cycle and notify once per file/ambiguity
state change.

**G11. No-op write guards.** `retireClosedTaskReferencesNow` (`:3322-3323`) and `processVaultFileText` (`:3190-3229`)
run `vault.process` over every markdown file even when nothing changes (mtime churn, modify-event feedback risk).
Prefilter with `cachedRead` + content checks (`![[` / target ids) and skip files where the transform is identity.

**G12. Improvements:** deduplicate the three identical status predicates (`:259-300`) whose drift caused the
11d08d9/ed22ddf fix series; fix `BLOCK_ID_RE.test(identity.blockId)` string-coercing `undefined` (`:3233`); hoist the
per-file `resolutions` map out of the migration loop (migrate-task-dependency-identities.mjs `:303-315`); preserve
per-line endings in the migration writer (`:289`) instead of forcing the detected newline; warn distinctly when the
migration's positional "link-order" fallback (`:186-188`) maps an unmatched id.

## Testing & verification

- **bob-cli:** every fix lands with a regression test (unit or tests/cli.rs / tests/tasks_parity.rs as appropriate — the
  reproduced scenarios above are the test cases). Full `cargo test` must pass, including
  `tests/tasks_real_vault_parity.rs`. Use the justfile's existing check/test targets if present.
- **bob-plugins:** extend `scripts/test-navigation-hotkeys.cjs`, `scripts/test-task-status-cycler.cjs`,
  `scripts/test-block-id-prompt.cjs`, `scripts/test-task-dependency-identity-migration.cjs` with regression cases; all
  suites must pass. Bump plugin `manifest.json` versions per repo convention. Deploy to the vault afterwards with
  `bob plugins sync` per the repo README.
- Workstreams A–G are independent and can be implemented/reviewed in parallel; C6's shared fence helper is the only
  cross-workstream code dependency (A3, B3 consume it — implement it first or duplicate minimally and consolidate at the
  end).

## Explicitly deferred (decisions or feature-sized work — not in this plan)

1. **Dependency-ID encoding for arbitrary paths** (spaces/dots/unicode) across bob-cli and both plugins — design
   decision; this plan only makes failures graceful.
2. JS-regex features (lookaround/backreferences) in `regex matches` — needs JS-engine regexes.
3. chrono-node-style free-text dates (`25th May 2023`, `in two weeks`, `14 October`).
4. `task.outlinks` / `task.file.tags` / `file.outlinks` population and `TasksDate.category`/`fromNow` shims in the
   QuickJS sandbox.
5. `backlink includes` — Tasks rejects backlink filtering outright; bob currently evaluates it against a different text
   form. Rejecting could break existing queries; needs a parity decision.
6. block-id-prompt: tasks with explicit `[id::]` are no longer resolvable by their bare `^block-id` (main.js:743-748) —
   possibly intended by "avoid ambiguous explicit-ID lookups"; needs intent confirmation.
7. collect_done: literal `__`-named files can silently collide with archive-qualified ids (false-negative ambiguity
   check); `.MD`-cased extensions in the test-helper grammar.
8. Multi-`## Pomodoros`-section scanning regression (only the first section is scanned now); capture symlink-replacing
   rename; `nearby_child_bullet_indentation` grandchild-indent pick; unifying the capture-vs-status "current Pomodoro"
   definition.
9. Tasks-parity leniency divergences (lowercase boolean operators accepted, comments bypassing placeholder expansion,
   `TQ_extra_instructions` ordering, projects frontmatter first-vs-last duplicate-key handling).
