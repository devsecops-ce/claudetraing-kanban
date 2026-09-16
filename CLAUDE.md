# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page **executive dashboard + Kanban board** tracking the readiness of every agent
skill available to this repository, built as a Claude Code training/demo artifact. Everything
lives in one file: `index.html` (markup, one `<style>` block, one `<script>` block).

The board was originally a fictional "UOB IT PMO" project board and was re-domained to a
skills catalog. The engine is unchanged; the *content* and the display labels are new.
**Internal field names are still the Kanban ones and deliberately differ from the UI labels:**

| Internal field | Shown in the UI as |
|---|---|
| `project`  | Source (where the skill came from) |
| `category` | Domain (what the skill is for) |
| `assignee` | Maintainer |
| `priority` | Relevance to this repo (values unchanged, so the `.p-*` / `.pill-*` CSS still applies) |
| `dueDate`  | Target ready (the date the skill should be usable by) |

Renaming those fields would be a large, low-value diff — but never assume a label and a field
name match when grepping.

Seed data is meant to be **true of this repo**: `image-3d` really did fail to install,
`persona-project-manager` really does need a `gws` binary that is not on PATH. If the real
state changes, change the seed rather than letting it drift into fiction.

## Hard constraints

These are the point of the exercise, not incidental. Breaking any one of them defeats the
deliverable, and several are easy to violate by reflex:

- **Vanilla HTML/CSS/JS only.** No React/Vue/jQuery/Tailwind, no bundler, no npm, no build step.
- **One file.** All markup, styles and script stay in `index.html`. Do not split out `.css`/`.js`.
- **Runs from `file://`.** It must work by double-clicking. No server, no ES modules, no `import`.
- **Zero external resources.** No CDN, no Google Fonts, no image files. System font stack;
  inline SVG or Unicode glyphs for icons. The only URL anywhere in the file is the FormSubmit
  endpoint.
- **No persistence of any kind.** No `localStorage`, `sessionStorage`, IndexedDB, or cookies.
  A refresh resetting the board to seed data is *intended behaviour* and is surfaced in the UI
  by the `.demo-note` banner — keep that banner honest if this ever changes.
- **No `alert()` or `confirm()`.** Validation uses inline `.field-error` text; delete uses an
  inline Yes/No row driven by a `confirmingDelete` flag in state.
- **No `!important` in CSS.** Palette and spacing come from custom properties on `:root`.

## Architecture

The script is one IIFE in `index.html`, organised under banner comments (`CONFIG`, `CONSTANTS`,
`STATE`, `HELPERS`, `FILTERING`, `RENDERING`, `STATE MUTATIONS`, `TOASTS`, `FORMSUBMIT
NOTIFICATION`, `FORM VALIDATION`, `MODAL`, `SUBMIT`, `EVENT WIRING`, `SEED DATA`, `INIT`).
Grep those banners to navigate.

**One-way data flow.** `state = { tasks: [], filters: {} }` is the single source of truth.
Every mutation (`addTask`, `moveTask`, `deleteTask`, `setDeleteConfirm`) edits the array and
then calls `renderBoard()`, which rebuilds all four columns from scratch via `board.innerHTML`.

Two invariants follow from that, and both matter:

1. **`renderBoard()` is the only place card DOM is written.** Never patch a card's text,
   badge or count in place — change state and re-render. A "small targeted DOM tweak" will be
   silently erased by the next render and will drift from state.
2. **All event handlers are delegated** onto the `#board` container (`click`, `change`,
   `dragstart`/`dragover`/`dragleave`/`drop`/`dragend`). Per-card listeners cannot work here —
   re-rendering destroys the nodes they were bound to. Card controls are dispatched by
   `data-action` + `data-id` attributes.

**Cards are built as HTML strings**, so every interpolated value must pass through
`escapeHtml()`. This is the one XSS boundary in the app; `title`, `description` and `assignee`
are free text straight from the user. When adding a field to `renderCard()`, escape it.

**`STATUSES`, `PROJECTS`, `CATEGORIES`, `PRIORITIES`** drive the columns, every `<select>`
(via `fillSelect()`), and `validateForm()`'s membership checks. Add an option in the constant
and it propagates; the only extra work for a new priority is a `.p-*` border-left rule and a
`.pill-*` rule in the CSS.

`STATUSES` is now `["Catalog", "Installed", "Blocked", "In Use"]`. **Renaming a column is not
just a string change** — the openModal/init defaults set `status` explicitly, and `isOverdue()`
excludes the terminal status. Both are covered below; grep for `SETTLED_STATUS` and for
`getElementById("status").value =`.

## The executive summary

`renderExecutive(visible)` is fed the **same filtered array** as `renderSummary()`, straight
from `renderBoard()`. That is the whole reason the stat tiles, the charts, the count badges and
the board can never disagree. Never give a chart its own copy of the data or its own filter
pass — one array, one render.

It renders four things, and the form of each is a deliberate choice, not decoration:

| Panel | Form | Why |
|---|---|---|
| Stat tiles | a bare number | a single figure is not a chart |
| Readiness | one horizontal **stacked bar** | part-to-whole. Never a pie or donut |
| Relevance, Domain | horizontal **bars**, one hue | magnitude, single series |
| Blocked | a **list** | "what is stuck and why" is not a magnitude question |

**Colour rules that are load-bearing:**

- Readiness is an **ordinal progression** (Catalog → Installed → In Use), so it uses one blue
  hue in three validated steps (`--viz-catalog` / `--viz-installed` / `--viz-inuse`), not four
  unrelated hues. The ramp passes monotone-lightness, adjacent ΔL ≥ 0.06, and a 2.50:1 light
  end on white. If you restep it, re-validate — do not eyeball it.
- **Blocked** is an exception state, so it takes the reserved status-critical red
  (`--viz-blocked`) and always ships with a ⚠ icon **and** a text label. Colour never carries
  the meaning alone, and that red is never reused as a series colour.
- Every chart also exposes a `<details>` data table. That is the documented relief for the
  lightest ramp step sitting under 3:1, so do not delete it to save space.

`"Usable today"` is `total - blocked`, **not** `Installed + In Use`. Catalog skills are built in
and usable on demand; counting only what is on disk understates the inventory by 16.

## Projection mode

`body.projection` is toggled by `#projectionBtn` and bumps a handful of type-scale custom
properties plus a short list of explicit overrides. It exists so the dashboard reads from the
back of a meeting room.

- The body class supplies the specificity, so it needs **no** priority overrides — keep it that
  way.
- It is in-memory like everything else here. It must not be persisted.
- `.projection-on` / `.projection-off` swap the button's own label; keep both in sync with
  `aria-pressed`.

## FormSubmit integration

`FORMSUBMIT_ENDPOINT` (top of `<script>`, under the `CONFIG` banner) is the only place the
recipient address appears. FormSubmit needs a **one-time activation**: the first POST triggers
a confirmation email to that address, and nothing is delivered until its link is clicked — so
the first submission is consumed by activation and will surface the failure toast.

`notifyNewTask()` is strictly optional to the app's function. `handleSubmit()` is optimistic
and must stay that way:

1. Validate; on failure render inline errors, keep the modal open, focus the first invalid field.
2. `addTask()` — the card is on the board *before* the network call.
3. Disable the submit button, label it "Sending…".
4. `await notifyNewTask()` in `try/catch`. Failure shows the warning toast
   ("Card added locally — email notification failed") and **keeps the card**.
5. `finally` re-enables the button, resets the form, closes the modal.

A FormSubmit failure must never break the board. Never send the recipient address anywhere
except this endpoint.

## Testing

There is no test runner, no linter, and no package manager here. Verification is manual plus
an ad-hoc Node harness.

The seed set is **24 skills** across all four columns. Counts on a clean load:
Catalog 16, Installed 3, Blocked 4, In Use 1, Overdue 2.

Constraint checks (fast, catches the most likely regressions):

```sh
grep -niE "localStorage|sessionStorage|indexedDB|document\.cookie|\balert\(|\bconfirm\(|!important" index.html
grep -noE "https?://[^\"' ]+" index.html   # expect only the FormSubmit endpoint
```

Note that the first grep also matches the *words* in comments, so a hit is not automatically a
violation — read the line. Do not write the literal token `!` + `important` into a comment; it
trips the project's own check.

Syntax check — extract the script block and parse it:

```sh
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' index.html > /tmp/app.js && node --check /tmp/app.js
```

Logic tests — the script is an IIFE with no exports, so to test internals, `sed` a
`module.exports = { ... }` line in after the `init();` call, then `require()` it under a
minimal fake `document`. The stub needs only `getElementById`/`createElement` returning objects
with `innerHTML`, `textContent`, `value`, `classList`, `addEventListener`, `setAttribute`,
`focus`, `reset`, `showModal`, `close`, `querySelector`/`querySelectorAll`, plus globals
`window.setTimeout` and a `fetch` stub. Then assert against `store.board.innerHTML` (card
counts, badges, overdue markers) and call `applyFilters()` / `validateForm()` / `moveTask()`
directly. This covers rendering, filtering, validation and escaping.

The stub also has to satisfy `renderExecutive()`, which reaches for `#kpiRow`,
`#readinessChart`, `#domainChart`, `#blockedPanel` and `#execSub` — a `getElementById` that
mints an object per id covers all of them.

What the harness cannot reach — check these in a real browser (`open index.html`): drag and
drop, the `<dialog>` modal, focus rings, the responsive stack below 768px, projection mode,
per-column scrolling, chart geometry, and the live network call.

Playwright's MCP server blocks `file:`, so to drive it use `python3 -m http.server` and load
`http://127.0.0.1:<port>/index.html`. The favicon 404 that produces is an artefact of serving
over HTTP and is not an app error.

## Behaviour details that are easy to get wrong

- **Dates are local `YYYY-MM-DD` strings**, produced by `todayIso()` / `daysFromToday()`.
  Do not reach for `toISOString()` for a calendar date — it shifts across the UTC boundary and
  makes "due today" read as overdue. ISO strings compare correctly with `<`, which is why
  `isOverdue()` is a plain string comparison.
- **Overdue excludes the terminal column.** A skill already in use has no pending target date,
  so it is never overdue. The rule reads `SETTLED_STATUS = STATUSES[STATUSES.length - 1]`
  rather than a hard-coded string — an earlier hard-coded `"Done"` silently broke this check
  the moment the columns were renamed. Seed data deliberately includes an `In Use` skill with a
  past target, so the correct on-load badge count is **2 out of 3** past-target skills — if a
  change makes it 3, that is the bug.
- **Seed dates are relative** (`daysFromToday(±n)`), so the board always looks current. Do not
  replace them with hard-coded dates.
- **The summary strip and count badges reflect the *filtered* view**, not the whole array —
  `renderSummary()` is fed the output of `applyFilters()`.
- **`dragleave` fires when crossing into a column's own children.** The handler guards against
  the resulting flicker with a `column.contains(event.relatedTarget)` check; keep it.
- **Drag and drop needs a keyboard equal.** Every card carries a labelled `Move ▸` select.
  Any new card interaction needs a non-pointer path too.
- Colour is never the only signal: priority pills carry their text, and the overdue badge says
  "Overdue".
