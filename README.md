# Claude Code Skills Catalog Board

An executive dashboard and Kanban board tracking the readiness of every agent skill
available to this repository. Built as a Claude Code training artifact: one file, no build
step, no dependencies.

**Live site:** https://devsecops-ce.github.io/claudetraing-kanban/

![The dashboard in the browser: a headline KPI row, an inventory-mix stacked bar, a domain
coverage chart, a blocked-skills callout, and a four-column Kanban board beneath.](docs/screenshot.png)

> Training artifact. Board state lives in memory only — refreshing resets it to the seeded
> inventory. Readiness reflects this repository at the time the seed data was written.

---

## What it shows

Twenty-four agent skills, each placed in one of four readiness states:

| Column | Meaning |
|---|---|
| **Catalog** | Built in and available on demand — nothing to install |
| **Installed** | Present in `.claude/skills` with prerequisites met |
| **Blocked** | Install failed, or a prerequisite is missing |
| **In Use** | Already used to build or ship this repository |

## Features

- **Executive summary** — four headline stat tiles (tracked / usable today / blocked /
  targets overdue), sized to read from the back of a meeting room.
- **Inventory mix** — a horizontal stacked bar for readiness plus a single-hue bar set for
  relevance, each with a legend and a `<details>` data table.
- **Coverage by domain** — horizontal bars, one hue, direct labels.
- **Blocked — needs a decision** — the ask, as a list rather than a chart: what is stuck and
  exactly why.
- **Projection mode** — one button scales the type and marks up for a projector.
- **Four-column board** with drag and drop, plus a keyboard-accessible `Move ▸` select on
  every card as an equivalent path.
- **Add Skill modal** with inline validation against the known sources, domains, relevance
  levels and readiness states.
- **Filtering** by source, relevance, and a free-text maintainer match. Charts, stat tiles,
  count badges and the board all read from the same filtered array, so they cannot disagree.
- **Inline delete confirmation** — a Yes/No row on the card, never a browser `confirm()`.

## Charts

Charts follow the data's job rather than decoration:

- Single figures are **stat tiles** — the number is the chart.
- Part-to-whole is **one horizontal stacked bar**, never a pie or donut.
- Magnitude is **horizontal bars**, one hue, direct labels.
- "What is stuck" is **a list**, because it is not a magnitude question.

Readiness is an ordinal progression (Catalog → Installed → In Use), so it uses one blue hue in
three validated steps rather than four unrelated colours. **Blocked** is an exception state, so
it takes the reserved status-critical red and always ships with a ⚠ icon and a text label —
colour never carries the meaning alone. The ramp was validated (monotone lightness, adjacent
ΔL ≥ 0.06, light end 2.50:1 on white), and every chart also exposes a data table.

## Tech stack

Vanilla HTML, CSS and JavaScript. No framework, no bundler, no npm, no CDN, no web fonts, no
image files — charts are laid out with CSS, not a charting library. System font stack; Unicode
glyphs for icons. The entire app is one `index.html`: one `<style>` block, one `<script>` IIFE.

## Getting started

```bash
git clone https://github.com/devsecops-ce/claudetraing-kanban.git
cd claudetraing-kanban
open index.html        # macOS — or just double-click the file
```

There is nothing to install and nothing to run. The page works from `file://`.

## Configuration

Skill-creation emails are off by default. To enable them, register the destination address at
[formsubmit.co](https://formsubmit.co), then replace the placeholder hash in `index.html`:

```js
const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/YOUR_FORMSUBMIT_HASH";
```

Use the hash from the activation email rather than a literal address — this file is served
publicly, so a plain address here is scrapeable and the open endpoint can be abused.

## Project structure

```
index.html           The entire application — markup, styles, script
CLAUDE.md            Guidance for Claude Code: constraints, architecture, invariants
story.md             How the project was built, and where the plan diverged from reality
README.md            This file
docs/screenshot.png  Screenshot of the live site, captured with Playwright
```

## Architecture notes

`state = { tasks: [], filters: {} }` is the single source of truth. Every mutation edits the
array and then calls `renderBoard()`, which rebuilds all four columns from scratch and then
feeds the same filtered array to `renderSummary()` and `renderExecutive()`. Three invariants
follow:

1. `renderBoard()` is the only place card DOM is written — never patch a card in place.
2. All event handlers are delegated onto the `#board` container; per-card listeners would be
   destroyed by the next re-render. Card controls dispatch on `data-action` + `data-id`.
3. The charts are never fed their own copy of the data. One filtered array, one render pass.

Cards and charts are built as HTML strings, so every interpolated value passes through
`escapeHtml()`. That is the app's one XSS boundary — `title`, `description` and `assignee` are
free user text.

Note that the internal field names are the original Kanban ones and differ from what the UI
shows: `project` renders as **Source**, `assignee` as **Maintainer**, `dueDate` as **Target
ready**, `priority` as **Relevance**, `category` as **Domain**.

See `CLAUDE.md` for the full constraint list before making changes.

## Persistence

There is none, by design — no `localStorage`, `sessionStorage`, IndexedDB or cookies. Board
state lives in memory only, and refreshing resets it to the seeded inventory. Projection mode
is in-memory too, so it resets on refresh. The in-page notice says so; keep it honest if that
ever changes.

## Deployment

The site is served by GitHub Pages from the `master` branch root. Pushing to `master`
republishes it — no build step and no workflow involved.
