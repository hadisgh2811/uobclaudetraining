# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A training exercise, not a product. `index.html` is a standalone IT project-management
Kanban board built as a demo artefact for a fictional "UOB IT PMO". `Notes-8 Sep 2026.txt`
holds the original build specification (lines 37-137) — it is the source of truth for every
requirement, so read it before changing behaviour. The PDF is course material, not code.

The demo deliberately uses no real UOB logo, trademark or branding — only a text wordmark
and a generic corporate blue palette. Keep it that way.

## Running and verifying

There is no build, no bundler, no package manager, no test runner, and no git repository.
Open `index.html` directly (double-click, or `Start-Process .\index.html`) — it must work
from a `file://` URL with no server. Verification is manual in the browser:

- 8 seeded tasks render across all four columns; summary strip shows Total 8 and a
  non-zero Overdue count.
- Drag a card between columns; the target column highlights and both count badges update.
- Tab to `Move ▸` → Enter → Tab to a status → Enter (keyboard path must work end to end).
- Submit the Add Task form empty: inline errors, no dialog. Past due date: date error.
- Add a task titled `<img src=x onerror=alert(1)>` — it must render as literal text.
- Refresh: the board returns to the 8 seeds.

## Hard constraints (do not break these)

These are the point of the exercise, and several are invisible unless you look for them:

- **Single file.** All markup, one `<style>` block, one `<script>` block in `index.html`.
  No splitting into `.css`/`.js` files.
- **No frameworks, no build step, no npm.** Vanilla HTML/CSS/JS only.
- **No external resources.** No CDN, no web fonts, no image files. System font stack;
  inline SVG or Unicode glyphs for icons.
- **No persistence of any kind.** No `localStorage`, `sessionStorage`, IndexedDB or
  cookies. Losing the board on refresh is intended behaviour and the toolbar carries a
  note saying so — if you ever add persistence, remove that note too.
- **FormSubmit is the only backend**, via its AJAX JSON endpoint (never a form POST, so
  the page never navigates). Its failure must never affect the board.
- **No `alert()`, no native `confirm()`.** Validation errors are inline text under each
  field; delete confirmation is an inline Yes/No block inside the card.
- **No `!important`** anywhere in the CSS.
- Colour is never the only signal — priority pills carry their label as text.

## Architecture of `index.html`

**Single source of truth.** `state = { tasks, filters, ui, nextIdNumber }` (~line 784).
`state.ui` holds transient interface flags (`moveMenuFor`, `confirmDeleteFor`, `focusKey`)
rather than those living in the DOM, so every interaction is: mutate `state` → call
`renderBoard()`. Nothing outside `renderBoard()`/`renderCard()` produces or edits card
markup. Adding a feature means adding to `state` and reading it in the renderer — not
patching DOM nodes in an event handler.

**Render path.** `renderBoard()` rebuilds the whole board's `innerHTML` from
`applyFilters()`, then calls `renderSummary()` and `restoreFocus()`. Because a full rebuild
destroys focus, interactive elements that must survive a re-render carry a `data-fk="..."`
focus key; a handler sets `state.ui.focusKey` before re-rendering and `restoreFocus()`
re-focuses that element. This is what keeps the keyboard move/delete flows usable — if you
add a control that appears as a result of a re-render, give it a `data-fk` too.

**Escaping.** All card and summary markup is built as strings, so every interpolated value
goes through `escapeHtml()`. There is no exception to this; a raw interpolation is a bug.

**Events are delegated**, not bound per card — one `click` listener on `#board` routes on
`data-action` (`open-move`, `close-move`, `move-to`, `delete-ask`, `delete-no`,
`delete-yes`), and the HTML5 drag listeners (`dragstart`/`dragover`/`dragleave`/`drop`) are
likewise on `#board`. Per-card listeners would leak on every re-render.

**Dates are compared as `YYYY-MM-DD` strings**, never as `Date` objects — `todayISO()`,
`isOverdue()` and the due-date validation all rely on lexicographic comparison to avoid
timezone drift. `formatDate()` parses with a regex for the same reason.

**Optimistic submit.** `handleSubmit()` validates, then calls `addTask()` and renders the
card *before* awaiting `notifyNewTask()`. The fetch is wrapped in try/catch and can only
change which toast appears — never whether the card exists. The submit button is disabled
with a `Sending…` label for the duration.

**Configuration lives in constants at the top of the script**: `FORMSUBMIT_ENDPOINT`
(~line 747, the only place the notification email address appears — never send it anywhere
else) and the `STATUSES`, `STATUS_META`, `PROJECTS`, `CATEGORIES`, `PRIORITIES` arrays that
drive the columns, the filter selects and the form selects alike. Both the board and the
form are generated from these, so adding a project or category is a one-line change.

FormSubmit needs a **one-time activation** per address: the first submission triggers a
confirmation email, and nothing is delivered until that link is clicked. With the
placeholder address in place the fetch simply fails and the warning toast fires — that is
the expected default state, not a bug.

**CSS** uses custom properties for the palette and a `--s-1`…`--s-6` spacing scale, both
defined in the `DESIGN TOKENS` block at the top. Numbered banner comments (1–10) divide the
stylesheet; the script uses the same banner convention. Columns are a 4-up CSS grid that
collapses to one column below 768px.
