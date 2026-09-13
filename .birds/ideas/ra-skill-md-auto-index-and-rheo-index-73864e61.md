---
id: ra-skill-md-auto-index-and-rheo-index-73864e61
short-id: '7'
title: 'SKILL.md: auto_index and rheo-index'
priority: 1
labels:
- feat-auto-index
deps: []
closed: false
---
`SKILL.md` is what teaches an agent rheo's author-facing surface, and it will be
wrong the moment `[spine] auto_index` lands in rheo.

That change, filed in `../rheo` under the same label: a content directory with
no landing file (`index.typ`, or `<dirname>.typ`) produces no page at all today
— it is a non-clickable group node — and will instead get a synthesized landing
page, DEFAULT ON. That page's whole body is a call to a new injected Typst
function, `rheo-index()`, which lists the directory's child pages as links. A
project replaces the listing by binding its own `rheo-index` in its
`[spine] prelude` file; the project's prelude is prepended after rheo's own
injection, so a later binding wins.

Without this update an agent reading `SKILL.md` goes on writing an `index.typ`
per directory whose only content is a hand-maintained list of the files beside
it — the exact work the key removes.

Touches: SKILL.md

## Steps

1. `SKILL.md`, the spine section at lines 80-136 (`## Spines — directory-scan
   default, exclude, include, sections`) — document `auto_index` beside the keys
   already there:
   - a directory with no landing file gets a page of its own, listing its
     children, and this is the DEFAULT;
   - `index.typ` and `<dirname>.typ` are both still landing files, and a
     directory carrying either is untouched;
   - `auto_index = false` restores the old behaviour, where such a directory has
     no page at all;
   - a directory left empty by `exclude` is still dropped entirely.

   Say explicitly that an agent should NOT write an `index.typ` whose only job
   is to list the directory's own contents. That is the instruction this file
   exists to give, and it is the one an agent will otherwise get wrong.

2. `SKILL.md`, the rheo.toml key list at lines 53-73 — add `auto_index` if that
   section enumerates `[spine]` keys as well; if it does not (the spine keys
   live in their own section at 80-136), leave it alone rather than starting a
   second list that can drift from the first.

3. `SKILL.md`, the Typst-surface section at lines 384-573 (`## Cross-file
   references`, through the `### rheo-context()` subsection ending at 573) — add
   `rheo-index()`: what it renders, that it reads the recursive `spine` tree
   (which includes directories) rather than `spine-flat` (which excludes them),
   and the override pattern, with the two-line example of binding it in a
   project's `[spine] prelude`.

## Non-goals

- Do NOT restate rheo's default listing markup. It is a plain list of links, and
  an agent that copies pinned HTML out of a skill will be wrong the first time
  that markup is tuned.
- Do NOT document a per-directory opt-out. There is none.
- Do NOT reorganize `SKILL.md`'s sections. Two insertions and, at most, one
  key added to an existing list.
- Do NOT touch any other repo. The engine change, the integration case and the
  docs site each have their own bird under the same label.

## VERIFY

There is no build and no test suite in this repo — the check is a prose review:

1. `SKILL.md` renders as valid markdown, headings and code fences intact.
2. Read the spine section end to end: the `auto_index` paragraph sits with the
   other spine keys and does not contradict the `exclude`/`include`/`section`
   text around it.
3. An agent following the file would now NOT create an `index.typ` for the sole
   purpose of listing a directory's contents, and WOULD know where to put a
   custom listing.