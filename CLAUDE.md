# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file HTML resume dashboard (`resume_dashboard1.html`): an editable "database" of resume
data on the left, a live print-ready preview on the right. No backend, no build step, no
dependencies — everything (HTML, CSS, JS) lives in one file.

## Commands

There is no build/lint/test tooling. The only "run" step: open `resume_dashboard1.html` directly
in a browser (double-click it, or `start resume_dashboard1.html` from this directory). Changes are
tested by reloading the file and checking the dashboard + print preview (`window.print()` /
Ctrl+P) in-browser.

## Architecture

**Data model** — each resume section is either a *singleton*, a *shared selectable list*, or a
*generic list*, all persisted to `localStorage` under keys prefixed `resume_`:

- **Profile** — singleton string (`DEFAULT_PROFILE`), edited via a single textarea.
- **Projects / Certifications** — these are the two *shared* sections. `masterProjects` /
  `masterCertifications` (`resume_projects_master` / `resume_certifications_master`) hold the
  actual content (title, bullets, tech stack, links / name, issuer) and are visible from every
  template — adding, editing, or deleting one affects all templates. What *does* vary per
  template is which ids are checked and their order, stored in each template's
  `data.projectSelection`/`data.projectOrder` (and `certSelection`/`certOrder`), not in the
  content itself. `projects` / `certifications` are the derived, per-template *working* arrays
  (`rebuildProjectsFromTemplate()` / `rebuildCertsFromTemplate()`, via `buildWorkingList()`) that
  the dashboard list, checkboxes, reorder buttons, and resume preview actually read from — treat
  them as a view, not a source of truth. Toggling a checkbox or reordering calls
  `persistProjectSelection()`/`persistCertSelection()`, which writes straight into the *active*
  template (no explicit save step, same autosave behavior as everywhere else in the app).
- **Generic sections** — Education, Experience, Technical Skills, Associations, Languages
  (`DEFAULT_EDUCATION`, `DEFAULT_EXPERIENCE`, `DEFAULT_SKILLS`, `DEFAULT_ASSOCIATIONS`,
  `DEFAULT_LANGUAGES`). These share one driver: the `SCHEMAS` object declares each section's
  fields (`text` or `lines` type), and `openGenericForm`/`saveGenericForm`/`renderGenericList`
  build the form and list UI from that schema instead of hand-written markup per section. Unlike
  Projects/Certifications, these are NOT shared — each template stores its own full copy in
  `data.education`/`data.experience`/etc., swapped in wholesale on template switch/update.

To add a new resume section, follow the generic-list pattern (add a `DEFAULT_*` array, a
`SCHEMAS` entry, a `genericStores` key, a collapsible `<details class="coll">` block in the
dashboard, and an output block in the preview) rather than the Projects/Certifications
shared-list pattern, unless the section needs checkboxes/reordering/a per-item custom form —
and if it does, decide deliberately whether its content should be shared across templates
(like Projects) or vary per template (like everything else).

**Templates** — named, switchable snapshots (`templates`, each `{ id, name, data }`,
persisted to `resume_templates`; `activeTemplateId` tracks which one is loaded). Switching
(`loadTemplate`) or deleting the active one applies the target's `data` via `applySnapshot()`:
profile/education/experience/skills/associations/languages are copied in wholesale, while
Projects/Certifications are rebuilt from the shared master lists filtered/ordered by that
template's selection. `saveAsNewTemplate()`/`updateCurrentTemplate()` do the reverse via
`currentSnapshot()`. `migrateLegacyTemplates()` runs once on load to upgrade any
localStorage written by the earlier (buggy) format, where each template carried a full
independent copy of `data.projects`/`data.certifications` — that caused a real bug where a
project added under one template silently didn't exist under another. Don't reintroduce
full per-template project/cert arrays; content belongs in the master lists only.

**Render loop** — `loadAll()` hydrates everything from `localStorage` (falling back to the
`DEFAULT_*` constants and migrating legacy template data), `renderAll()` re-renders both the
dashboard lists and the resume preview (`renderResumePreview()`) any time data changes. There's
no framework/virtual DOM: every mutation function (`toggleSelected`, `moveProject`,
`saveGenericForm`, etc.) persists then calls `renderAll()` directly, rebuilding the relevant
`innerHTML` from the in-memory arrays.

**Export/import** — every section has a per-section JSON export/import (`exportSection`/
`importSection`, matching the top-level `*.json` files in this repo) plus a combined
`exportAll`/`importAll` (`resume_data_full.json`). `projects.json`/`certifications.json` export
the shared master lists (not any one template's filtered view). The per-section JSON files at
the repo root are seeded defaults / backups — they are not read at runtime; the `DEFAULT_*`
constants in the `<script>` are the actual source of truth for first load.

**Print/PDF** — the `@media print` block hides `.dash`/`.no-print` and reflows `.preview-wrap` /
`.page` to fill the page; `window.print()` (bound to the "Print / Save as PDF" button) is how a
final resume gets produced. Layout is deliberately single-column and avoids `position:
relative`/`absolute` entirely — right-aligned dates/locations use flexbox `space-between`, bullet
markers use `::marker`. This isn't a style preference: Chromium's PDF text export writes
positioned elements *after* all normal-flow content in their stacking context, so anything
absolutely positioned (the old right-aligned date columns, the old header contact column) got
scrambled to the end of the extracted text, detached from its job title/section — verified via
headless print-to-PDF + text extraction. Don't reintroduce `position: relative`/`absolute` in the
resume/preview output for layout purposes; it silently breaks ATS text parsing even though it
looks fine on screen and in the rendered PDF.

## Repo layout notes

- `Resume/`, `coverletters/`, and `resume_dashboard.html` are git-ignored (see `.gitignore`) —
  personal output files and a superseded earlier version of the tool, not part of what's shipped.
  Don't add tracked files there, and don't treat `resume_dashboard.html` as canonical.
- Other top-level `.html` files besides `resume_dashboard1.html` (if present) are typically WIP
  variants exploring new sections before they're folded into the main file — check with the user
  before treating them as canonical.
