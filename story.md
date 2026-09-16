# Build Story — UOB IT PMO Kanban Board

A record of how this project was built in a single session with Claude Code, on 16 September 2026.
It covers what was built, how it was verified, and — more usefully — the three points where
reality diverged from the plan.

This is a training artifact. The app is a demo, not a real UOB system.

**Live site:** https://devsecops-ce.github.io/claudetraing-kanban/

---

## The brief

A single-page IT project management board for a fictional internal "UOB IT PMO", under
deliberately tight constraints:

- Vanilla HTML/CSS/JS only — no React, no bundler, no npm, no build step
- One file, `index.html`, runnable by double-clicking it
- No persistence of any kind: no `localStorage`, `sessionStorage`, IndexedDB or cookies
- No external resources — no CDN, no web fonts, no image files
- Form submissions go out through FormSubmit's AJAX endpoint; no other backend
- No `alert()` or `confirm()`; no `!important`

The constraints are the exercise. Most of them are easy to violate by reflex, which is why they
were later written into `CLAUDE.md` rather than left implicit.

---

## Stage 1 — Planning

The working directory held a PDF, a notes file, and an empty `Kanbanboard/` folder created
minutes earlier. No git, no existing code, nothing to reuse.

Three decisions weren't inferable from the brief, so they were put to the user rather than
guessed:

| Question | Chosen |
|---|---|
| Where should `index.html` live? | `Kanbanboard/` |
| Which address should FormSubmit notify? | The user's training mailbox |
| Add Task form: modal or sidebar? | Modal, opened from the header |

A written plan followed — palette tokens, the state/render architecture, the seed data, and the
verification approach — and was approved before any code was written.

---

## Stage 2 — Building

The result is one file of roughly 1,370 lines: markup, a single `<style>` block, and a single
`<script>` IIFE organised under banner comments (`CONFIG`, `CONSTANTS`, `STATE`, `HELPERS`,
`FILTERING`, `RENDERING`, `STATE MUTATIONS`, `TOASTS`, `FORMSUBMIT NOTIFICATION`,
`FORM VALIDATION`, `MODAL`, `SUBMIT`, `EVENT WIRING`, `SEED DATA`, `INIT`).

### The architectural decision everything else follows from

`state = { tasks, filters }` is the single source of truth. Every mutation — `addTask`,
`moveTask`, `deleteTask`, `setDeleteConfirm` — edits the array and then calls `renderBoard()`,
which rebuilds all four columns from scratch.

Two consequences, both non-obvious and both load-bearing:

1. **`renderBoard()` is the only place card DOM is written.** A "quick targeted DOM tweak"
   elsewhere gets silently erased by the next render and drifts from state.
2. **Every handler is delegated** onto the board container. Per-card listeners cannot work,
   because re-rendering destroys the nodes they were bound to. Card controls dispatch on
   `data-action` + `data-id` attributes instead.

Because cards are assembled as HTML strings, every user-supplied value passes through
`escapeHtml()`. That function is the app's single XSS boundary — `title`, `description` and
`assignee` are free text typed by the user.

### Details that are easy to get wrong

- **Dates are local `YYYY-MM-DD` strings.** Using `toISOString()` for a calendar date shifts
  across the UTC boundary and makes "due today" read as overdue.
- **Overdue excludes `Done`.** The seed data deliberately includes a completed task with a past
  due date, so the correct badge count on load is 2 of 3 past-due tasks. If it ever shows 3,
  that's the bug.
- **`dragleave` fires when the cursor crosses into a column's own children,** which causes the
  drop highlight to flicker. The handler guards with a `column.contains(event.relatedTarget)`
  check.
- **Drag-and-drop needs a keyboard equal.** Every card carries a labelled `Move ▸` select, so
  the board is fully operable without a mouse.

---

## Stage 3 — Verification

The build was checked three ways rather than by eyeballing the browser.

**1. Constraint greps** — confirming the rules actually held:

```sh
grep -niE "localStorage|sessionStorage|indexedDB|document\.cookie|\balert\(|\bconfirm\(|!important" index.html
grep -noE "https?://[^\"' ]+" index.html   # expect only the FormSubmit endpoint
```

**2. Syntax check** — extract the script block and parse it:

```sh
awk '/^<script>$/{f=1;next} /^<\/script>$/{f=0} f' index.html > /tmp/app.js && node --check /tmp/app.js
```

**3. A logic harness.** The script is an IIFE with no exports, so a `module.exports = {...}` line
was `sed`-ed in after `init()`, and the file `require`d under a minimal fake `document` — a stub
providing only `getElementById`, `createElement`, and objects with `innerHTML`, `value`,
`classList`, `addEventListener` and friends, plus a `fetch` stub.

That ran **26 assertions**, all passing: XSS escaping, ID zero-padding, move and delete, all
three filters and their intersection, overdue edge cases, and every validation boundary
(80-character title, 500-character description, past vs. today's date).

The first render was confirmed as 4 columns, 8 cards, badges `2, 3, 2, 1`, and 2 overdue markers.

**What the harness cannot reach** — drag-and-drop, the `<dialog>` modal, focus rings, the
responsive stack below 768px, and any real network call. Those need a browser.

---

## Stage 4 — Documentation

`CLAUDE.md` was generated for future sessions. A scan found no existing README, Cursor or
Copilot rules, or other agent configs to carry over.

It deliberately records what *isn't* discoverable by reading one file: the hard constraints, the
two render invariants above, the FormSubmit contract, how to rebuild the test harness, and the
date-handling traps. It contains no file tree and no generic development advice.

---

## Stage 5 — Deploying, and where the plan broke

The deploy is where the interesting failures live. All three were caught by checking rather than
assuming.

### Failure 1 — FormSubmit rejects `file://` outright

Triggering the activation email returned:

> `{"success":"false","message":"Make sure you open this page through a web server, FormSubmit will not work in pages browsed as HTML files."}`

A second request with a web-origin `Referer` header succeeded, isolating the cause precisely.

**This contradicted the brief.** The app was required to run from a double-clicked file *and* to
submit through FormSubmit — but FormSubmit refuses `file://` origins, so notifications were never
going to work locally. Deploying is what makes them work.

The optimistic-submit design absorbed this without a change: the card is added to the board
*before* the network call, and a failure only produces a warning toast. The board never depends
on the request succeeding. A constraint conflict that could have been a rewrite turned out to be
a comment update, purely because the failure path was designed in from the start.

### Failure 2 — a repo already existed, with the email already public

`git init` printed `re-init: ignored --initial-branch=main`, and `index.html` showed as
*modified* rather than *added*. Both are signals that a repo was already there — the user had
created and pushed `claudetraing-kanban` on branch `master` in parallel.

Inspecting rather than committing over it revealed the real problem: the already-pushed commit
`1b31529` contained the literal mailbox address at line 723, on a **public** repository. That
directly contradicted the user's stated choice to keep the address out of published source.

The fix had to be a history rewrite, not a new commit — adding a commit on top leaves the
address fully readable in the previous one. With exactly one commit in the repo, amending it was
clean.

**The residue is worth stating plainly:** GitHub keeps unreachable objects fetchable by SHA for a
period after a force-push. Anyone who saw the original commit hash can still retrieve it. Fully
closing that would mean deleting and recreating the repository. The address should be treated as
harvested.

### Failure 3 — the push was rejected for a missing OAuth scope

The token carried `read:org, repo` but not `workflow`, so pushing the Actions workflow failed:

> `refusing to allow an OAuth App to create or update workflow .github/workflows/deploy.yml without workflow scope`

GitHub rejects the **entire push** on this, which meant the email fix was blocked along with the
workflow. The sequencing mattered: the workflow file was dropped from the commit and the fix
pushed on its own, closing the exposure immediately, with the workflow deferred.

Pages was then enabled from the `master` branch instead of Actions. Setting the Pages build type
to `workflow` with no workflow present would have served a 404 — branch-based deployment gets a
working site now and converts to Actions later without downtime.

---

## Outcome

Live at https://devsecops-ce.github.io/claudetraing-kanban/, deployed from `master`, commit
`60dd2a2`.

Verified after deployment by fetching the site and checking the served bytes: HTTP 200 over
HTTPS, correct `<title>`, all four column names present, and zero occurrences of the mailbox
address.

### Outstanding

- **Convert to GitHub Actions.** The workflow is written at `.github/workflows/deploy.yml` but
  untracked. It needs `gh auth refresh -s workflow`, then a commit and a switch of the Pages
  build type.
- **Restore notifications.** `FORMSUBMIT_ENDPOINT` currently points at a placeholder. FormSubmit's
  activation email supplies a random alias endpoint (`formsubmit.co/ajax/<hash>`) which works
  identically without revealing an address — the right value for a public repo.

---

## What generalises

**Design the failure path first.** The optimistic submit pattern — render locally, then notify,
and treat notification failure as cosmetic — meant an unexpected platform restriction cost a
comment instead of a redesign.

**Treat surprising tool output as a signal, not noise.** `re-init` and a file showing as
*modified* instead of *added* are each a single word of difference. Following them up found a
live credential exposure.

**Verify what was served, not what was pushed.** A successful `git push` and a successful Pages
build both mean very little on their own. Fetching the deployed URL and grepping the response is
the check that actually confirms the outcome.

**Ordering matters when a fix is blocked.** The scope rejection bundled a security fix with a
feature. Splitting them shipped the fix in minutes rather than after an interactive
re-authentication.
