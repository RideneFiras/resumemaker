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

**Data model** — each resume section is either a *singleton*, a *selectable list*, or a
*generic list*, all persisted to `localStorage` under keys prefixed `resume_` (e.g.
`resume_projects`, `resume_education`):

- **Profile** — singleton string (`DEFAULT_PROFILE`), edited via a single textarea.
- **Projects** — selectable/reorderable list (`DEFAULT_PROJECTS`). Each item has a `selected`
  checkbox (only checked projects render on the resume), up/down reorder buttons, and its own
  add/edit form (`#editForm`) with dedicated fields (title, bullets, tech stack, GitHub/YouTube
  links). This is the only section with custom form markup because of the optional link icons.
- **Generic sections** — Education, Experience, Technical Skills, Associations, Languages
  (`DEFAULT_EDUCATION`, `DEFAULT_EXPERIENCE`, `DEFAULT_SKILLS`, `DEFAULT_ASSOCIATIONS`,
  `DEFAULT_LANGUAGES`). These share one driver: the `SCHEMAS` object declares each section's
  fields (`text` or `lines` type), and `openGenericForm`/`saveGenericForm`/`renderGenericList`
  build the form and list UI from that schema instead of hand-written markup per section.

To add a new resume section, follow the generic-list pattern (add a `DEFAULT_*` array, a
`SCHEMAS` entry, a `genericStores` key, a collapsible `<details class="coll">` block in the
dashboard, and an output block in the preview) rather than the one-off Projects pattern, unless
the section needs Projects-style checkboxes/reordering/custom links.

**Render loop** — `loadAll()` hydrates all sections from `localStorage` (falling back to the
`DEFAULT_*` constants), `renderAll()` re-renders both the dashboard lists and the resume preview
(`renderResumePreview()`) any time data changes. There's no framework/virtual DOM: every mutation
function (`toggleSelected`, `moveProject`, `saveGenericForm`, etc.) calls `save*()` then
`renderAll()` directly, rebuilding the relevant `innerHTML` from the in-memory arrays.

**Export/import** — every section has a per-section JSON export/import (`exportSection`/
`importSection`, matching the top-level `*.json` files in this repo) plus a combined
`exportAll`/`importAll` (`resume_data_full.json`). The per-section JSON files at the repo root
(`profile.json`, `education.json`, `projects.json`, etc.) are the seeded defaults / backups — they
are not read at runtime; the `DEFAULT_*` constants in the `<script>` are the actual source of
truth for first load.

**Print/PDF** — the `@media print` block hides `.dash`/`.no-print` and reflows `.preview-wrap` /
`.page` to fill the page; `window.print()` (bound to the "Print / Save as PDF" button) is how a
final resume gets produced. Layout is deliberately single-column with absolute-positioned date/
location columns instead of grid/flexbox, to stay robust across PDF/ATS text parsers.

## Repo layout notes

- `Resume/`, `coverletters/`, and `resume_dashboard.html` are git-ignored (see `.gitignore`) —
  personal output files and a superseded earlier version of the tool, not part of what's shipped.
  Don't add tracked files there, and don't treat `resume_dashboard.html` as canonical.
- Other top-level `.html` files besides `resume_dashboard1.html` (if present) are typically WIP
  variants exploring new sections before they're folded into the main file — check with the user
  before treating them as canonical.
