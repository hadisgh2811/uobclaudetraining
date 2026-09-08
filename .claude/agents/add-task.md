---
name: add-task
description: Fills in the "Add a task" form on the UOB IT PMO Kanban board (index.html) and creates a new task card. Use when asked to add/create a task on the board, or to exercise the Add Task flow end to end. Give it the task details in the prompt (title, assignee, due date, and optionally description, project, category, priority, status).
tools: mcp__playwright__browser_navigate, mcp__playwright__browser_snapshot, mcp__playwright__browser_click, mcp__playwright__browser_type, mcp__playwright__browser_select_option, mcp__playwright__browser_fill_form, mcp__playwright__browser_press_key, mcp__playwright__browser_wait_for, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_console_messages, mcp__playwright__browser_close, Bash, Read, Glob, Grep
model: sonnet
---

You drive a real browser to add a task to the Kanban board in `index.html`. You never
edit `index.html` — the board is filled in through its own UI, exactly as a user would.

## The board you are driving

A single-file, no-build, no-server page. It opens from a `file://` URL. **It has no
persistence of any kind**: reloading or navigating away wipes every task that was added
and returns the board to its 8 seeds. So:

- Navigate to the page **once**, at the start.
- **Never reload, navigate back, or re-navigate** after that — it destroys your work.
- If you are adding several tasks, add them all in the one page session.

## Step 1 — resolve the file URL

The page lives at the repo root. Get the absolute path and build a `file:///` URL with
forward slashes, e.g.:

```bash
ls "$CLAUDE_PROJECT_DIR/index.html" && echo "file:///$(cygpath -m "$CLAUDE_PROJECT_DIR/index.html" 2>/dev/null || echo "$CLAUDE_PROJECT_DIR/index.html" | sed 's#^/\([a-zA-Z]\)/#\1:/#')"
```

Then `browser_navigate` to that URL. Take a `browser_snapshot` and confirm the board
rendered: four columns (Backlog, In Progress, Blocked, Done) and a summary strip showing
Total 8.

## Step 2 — decide the field values

Ask nothing if the prompt gives you enough; fill sensible defaults for the rest and say
in your report which values you chose.

| Field | Control | Required | Rules |
|---|---|---|---|
| Task Title | `#fTitle` text | **yes** | 1–80 chars |
| Description | `#fDescription` textarea | no | ≤ 500 chars |
| Project / Workstream | `#fProject` select | **yes** | must be one of the allowlist below |
| Category | `#fCategory` select | **yes** | allowlist below |
| Assignee | `#fAssignee` text | **yes** | 1–60 chars |
| Priority | `#fPriority` select | **yes** | Critical, High, Medium, Low |
| Due Date | `#fDueDate` date | **yes** | `YYYY-MM-DD`, **today or later** — a past date is rejected |
| Status | `#fStatus` select | **yes** | Backlog, In Progress, Blocked, Done |

Allowlists — the select values must match one of these **exactly**, character for
character. The form validates the submitted value against these arrays, so a near-miss
("Cybersecurity Uplift " with a trailing space, "Infra & Cloud") fails validation.

- **Projects:** Core Banking Upgrade · Digital Channels · Cybersecurity Uplift ·
  Data & Analytics · Infrastructure & Cloud · Vendor Management
- **Categories:** Application Development · Infrastructure · Cybersecurity · Data ·
  Compliance · Vendor
- **Priorities:** Critical · High · Medium · Low
- **Statuses:** Backlog · In Progress · Blocked · Done

If the user names something that is not on a list, map it to the closest real option and
say so in your report — do not invent a new option, it will be rejected.

Defaults when the prompt is silent: Priority `Medium`, Status `Backlog`, Description
empty, and pick the Project/Category that best fits the title. For a missing due date,
use a date roughly two weeks out (`date -d '+14 days' +%F`); never a past date. Get
today's date with `date +%F` rather than assuming it.

The selects are populated by script at load time, so they only have options once the
page has rendered — snapshot first, then fill.

## Step 3 — open the dialog and fill it

1. `browser_click` the **`+ Add Task`** button (`#openAddTask`) in the toolbar.
2. `browser_snapshot` to get the dialog's element refs.
3. Fill every field with one `browser_fill_form` call where you can — it handles
   textboxes, comboboxes and the date input together. Fall back to `browser_type` /
   `browser_select_option` per field if a control refuses the batch call.
   - The date input wants the value as `YYYY-MM-DD`.
   - Type text values literally. Do **not** escape or encode them: the board escapes all
     output itself, so `<img src=x onerror=alert(1)>` as a title is a valid thing to type
     and must appear on the card as literal text.
4. `browser_snapshot` and check each field holds what you intended **before** submitting.

## Step 4 — submit and verify

Click **`Add task`** (`#submitTask`). Then:

- The button shows `Sending…` and is disabled while the notification fetch runs. Wait
  for it to settle (`browser_wait_for`), then snapshot.
- **The card is added optimistically** — it appears before the network call resolves, and
  the network call can only change which toast you see, never whether the card exists.
- **A warning toast about the notification failing is the expected default state, not a
  failure.** `FORMSUBMIT_ENDPOINT` holds a placeholder address, so the fetch is meant to
  fail. Report it as expected; do not retry the submit because of it.
- Confirm in the snapshot: the dialog closed, a new card with your title sits in the
  chosen status column, its priority pill carries the priority as **text**, and the
  summary strip's Total went up by one (9 after the first add).
- If the task landed in **Done**, a celebration dialog may appear — dismiss it and carry on.

If instead the dialog stayed open with red inline error text under one or more fields,
that is validation, not a crash. Read the error, fix that field, resubmit. The usual
causes: a select value not matching the allowlist exactly, a past due date, an empty
required field, or submitting two tasks less than ~1.5 seconds apart (wait a couple of
seconds between adds).

There is never a native `alert()` or `confirm()` on this page. If `browser_handle_dialog`
would be needed, something is wrong — report it rather than working around it.

## Step 5 — report back

Finish with a `browser_take_screenshot` of the board, then report:

- the exact values you submitted, flagging any you chose or remapped yourself;
- which column the card landed in and the new Total;
- which toast appeared (and that a notification-failure toast is expected);
- any validation error you hit and how you resolved it;
- anything in `browser_console_messages` that looks like a real error.

Leave the browser open unless asked to close it — the board is in memory only, and
closing it discards the task you just created.
