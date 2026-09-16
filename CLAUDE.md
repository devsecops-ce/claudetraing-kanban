# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page Kanban board for a fictional internal "UOB IT PMO", built as a Claude Code
training/demo artifact. Everything lives in one file: `index.html` (~1370 lines: markup,
one `<style>` block, one `<script>` block).

Not a real UOB system. Use only the text wordmark "UOB IT PMO" and the generic corporate
blue palette — never a real UOB logo, trademark, or an imitation of an official UOB screen.

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

Constraint checks (fast, catches the most likely regressions):

```sh
grep -niE "localStorage|sessionStorage|indexedDB|document\.cookie|\balert\(|\bconfirm\(|!important" index.html
grep -noE "https?://[^\"' ]+" index.html   # expect only the FormSubmit endpoint
```

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

What the harness cannot reach — check these in a real browser (`open index.html`): drag and
drop, the `<dialog>` modal, focus rings, the responsive stack below 768px, and the live
network call.

## Behaviour details that are easy to get wrong

- **Dates are local `YYYY-MM-DD` strings**, produced by `todayIso()` / `daysFromToday()`.
  Do not reach for `toISOString()` for a calendar date — it shifts across the UTC boundary and
  makes "due today" read as overdue. ISO strings compare correctly with `<`, which is why
  `isOverdue()` is a plain string comparison.
- **Overdue excludes `Done`.** A completed task with a past due date is not overdue. Seed data
  deliberately includes such a task, so the correct on-load badge count is 2 out of 3 past-due
  tasks — if a change makes it 3, that is the bug.
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
