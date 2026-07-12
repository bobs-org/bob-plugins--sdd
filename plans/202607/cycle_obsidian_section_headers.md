---
create_time: 2026-07-12 13:08:26
status: done
prompt: 202607/prompts/cycle_obsidian_section_headers.md
tier: tale
---

# Plan: Cycle Obsidian section-header navigation

## Goal

Make the existing `Ctrl+J` and `Ctrl+K` Obsidian commands traverse real Markdown section headers cyclically within the
active note. Forward navigation should wrap from the end of the note to the first header, and backward navigation should
wrap from the beginning to the last header.

## Current behavior and scope

- The vault already binds `Ctrl+J` to `bob-navigation-hotkeys:jump-to-next-section-header` and `Ctrl+K` to
  `bob-navigation-hotkeys:jump-to-prev-section-header`; no hotkey configuration change is needed.
- `plugins/bob-navigation-hotkeys/main.js` already centralizes heading discovery in `getSectionHeaderLines`, excluding
  YAML frontmatter and fenced code blocks, and uses `getSectionHeaderJumpLine` to select the next strictly later or
  previous strictly earlier header.
- At a document boundary, `getSectionHeaderJumpLine` currently returns no target, causing the command to show a “No
  next/previous section header” notice. The source-of-truth change belongs in the `bob-plugins` repository, not in the
  deployed vault copy.

## Intended semantics

- With two or more valid headers, choose the nearest header strictly in the requested direction; if none exists, wrap to
  the opposite endpoint.
- When the cursor is on a header, move to the following/preceding header before wrapping, so repeated presses cycle in
  document order.
- With exactly one valid header, both directions resolve to that header, including when the cursor is already there.
- With no valid headers, preserve the existing no-target result and user notice.
- Preserve current invalid-cursor handling and current header recognition rules, including ignoring apparent headings in
  frontmatter or fenced code.

## Implementation

1. Update `getSectionHeaderJumpLine` in `plugins/bob-navigation-hotkeys/main.js` so a non-empty header list falls back
   to its first entry for forward navigation and its last entry for backward navigation after the strict directional
   search is exhausted. Keep `jumpToSectionHeader` responsible for cursor placement, top alignment, and no-header
   notices.
2. Add focused regression cases to `scripts/test-navigation-hotkeys.cjs` for normal forward/backward movement,
   wraparound at and beyond both document boundaries, cycling from a cursor positioned directly on a header, the
   single-header case, and the no-header case. Include frontmatter and fenced pseudo-headings in the fixtures so wrap
   targets are proven to use only real section headers.
3. Bump the Bob Navigation Hotkeys patch version in `plugins/bob-navigation-hotkeys/manifest.json` and keep the version
   shown in the repository `README.md` synchronized.

## Verification and deployment

1. Run the focused navigation-hotkeys test file, then the repository-wide `npm test` and `npm run validate` checks.
2. Deploy only the updated source-of-truth plugin with `bob plugins sync -p bob-navigation-hotkeys`.
3. Smoke-test in Obsidian with a multi-section note: repeated `Ctrl+J` must cycle first-to-last-to-first, repeated
   `Ctrl+K` must cycle last-to-first-to-last, and a note without valid headers must retain the existing notice instead
   of moving the cursor.
