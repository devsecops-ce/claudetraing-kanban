# UOB IT PMO Kanban Board

A single-page Kanban board for a fictional internal "UOB IT PMO", built as a Claude Code
training artifact. One file, no build step, no dependencies.

**Live site:** https://devsecops-ce.github.io/claudetraing-kanban/

> Demo / training tool only. Not an official UOB system; no real UOB branding or data is used.
> All tasks and assignee names in the board are fictional seed data.

---

## Features

- **Four-column board** — Backlog, In Progress, Blocked, Done, each with a short hint line
  and a live task count.
- **Drag and drop** between columns, with keyboard-accessible status `<select>` on every card
  as an equivalent path.
- **Add Task modal** with inline validation — required fields, a due date that must not be in
  the past, and membership checks against the known projects, categories and priorities.
- **Filtering** by project, priority, and a free-text assignee match, with a Clear filters reset.
- **Inline delete confirmation** — a Yes/No row on the card, no browser `confirm()` dialog.
- **Toast notifications** for add, move and delete.
- **Optional email notification** on task creation via FormSubmit's AJAX endpoint
  (disabled until you supply your own hash — see Configuration).
- **Priority and category pills** colour-coded from CSS custom properties.

## Tech stack

Vanilla HTML, CSS and JavaScript. No framework, no bundler, no npm, no CDN, no web fonts,
no image files. System font stack; inline SVG and Unicode glyphs for icons. The entire app is
~1,380 lines in `index.html`: one `<style>` block, one `<script>` IIFE.

## Getting started

```bash
git clone https://github.com/devsecops-ce/claudetraing-kanban.git
cd claudetraing-kanban
open index.html        # macOS — or just double-click the file
```

There is nothing to install and nothing to run. The page works from `file://`.

## Configuration

Task-creation emails are off by default. To enable them, register the destination address at
[formsubmit.co](https://formsubmit.co), then replace the placeholder hash in `index.html`:

```js
const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/YOUR_FORMSUBMIT_HASH";
```

Use the hash from the activation email rather than a literal address — this file is served
publicly, so a plain address here is scrapeable and the open endpoint can be abused.

## Project structure

```
index.html    The entire application — markup, styles, script
CLAUDE.md     Guidance for Claude Code: constraints, architecture, invariants
story.md      How the project was built, and where the plan diverged from reality
README.md     This file
```

## Architecture notes

`state = { tasks: [], filters: {} }` is the single source of truth. Every mutation edits the
array and then calls `renderBoard()`, which rebuilds all four columns from scratch. Two
invariants follow:

1. `renderBoard()` is the only place card DOM is written — never patch a card in place.
2. All event handlers are delegated onto the `#board` container; per-card listeners would be
   destroyed by the next re-render. Card controls dispatch on `data-action` + `data-id`.

Cards are built as HTML strings, so every interpolated value passes through `escapeHtml()`.
That is the app's one XSS boundary — `title`, `description` and `assignee` are free user text.

See `CLAUDE.md` for the full constraint list before making changes.

## Persistence

There is none, by design — no `localStorage`, `sessionStorage`, IndexedDB or cookies. Board
state lives in memory only, and refreshing resets it to the seeded demo data. The in-page
notice says so; keep it honest if that ever changes.

## Deployment

The site is served by GitHub Pages from the `master` branch root. Pushing to `master`
republishes it — no build step and no workflow involved.
