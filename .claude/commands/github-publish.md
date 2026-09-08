---
description: Scan for secrets, push to GitHub, then set up Pages via Actions, the README and the repo About link
argument-hint: [github repo URL or owner/repo — omit to reuse the existing origin remote]
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, WebFetch
---

# Publish this project to GitHub

Target repository: **$ARGUMENTS**

If that is empty, use the existing `origin` remote. If there is no `origin` either,
stop and ask the user for the repo URL — do not invent one and do not create a repo
under a guessed account name.

Accept either form and normalise to `owner/repo`:
- `https://github.com/OWNER/REPO` (with or without `.git`)
- `git@github.com:OWNER/REPO.git`
- `OWNER/REPO`

## Ground rules

- **The secret scan runs before the first push, not after.** Once a secret is pushed
  it is in the remote history and rotating the credential is the only real fix.
  The user's step list puts scanning last; do it first anyway and say so.
- **Never run `git push --force`, `git filter-branch`, or history rewrites** unless the
  user explicitly asks in this conversation.
- Never `git add -A` blindly. Review `git status` and stage deliberately.
- Pushing is outward-facing and hard to undo. Show the user what will be pushed and
  get confirmation before the first push to a new remote.
- Respect this repo's `CLAUDE.md` constraints when editing any project file.

## Step 0 — Preflight

Run these and report what you find:

```
git rev-parse --is-inside-work-tree
git status --porcelain=v1 -b
git remote -v
git branch --show-current
git log --oneline -5
```

Detect available tooling, in this order of preference:

1. `gh --version` and `gh auth status` — the GitHub CLI. Easiest path; use it for repo
   creation, the About/homepage edit and Pages settings.
2. No `gh`, but `$GITHUB_TOKEN` / `$GH_TOKEN` is set — use the REST API via `curl`.
3. Neither — plain `git` can still push over HTTPS (credential manager / PAT prompt),
   but the About and Pages *settings* cannot be set from the CLI. In that case do
   everything you can, then hand the user an exact click-path for the rest.

On Windows, `gh` may exist in PowerShell but not in the Bash tool's PATH — check both
before concluding it is missing.

If the repo has no commits yet, or `main` is not the current branch, sort that out
before pushing and tell the user what you changed.

## Step 1 — Scan for sensitive data (BEFORE pushing)

Scan the working tree *and*, if any commits exist, what is already committed.
Report findings as a short table: file:line, what it looks like, and your confidence.

Search for at minimum:

- **Credentials by pattern**: `api[_-]?key`, `secret`, `password`, `passwd`, `token`,
  `credential`, `private[_-]?key`, `client[_-]?secret`, `authorization:\s*bearer`
- **Known key shapes**: `AKIA[0-9A-Z]{16}` (AWS), `gh[pousr]_[A-Za-z0-9]{36,}` (GitHub),
  `sk-[A-Za-z0-9]{20,}` (OpenAI-style), `sk-ant-` (Anthropic),
  `AIza[0-9A-Za-z_-]{35}` (Google), `xox[baprs]-` (Slack),
  `-----BEGIN [A-Z ]*PRIVATE KEY-----`, JWTs (`eyJ[A-Za-z0-9_-]{10,}\.`)
- **Connection strings**: `(mongodb|postgres|postgresql|mysql|redis|amqp)://[^\s]*:[^\s]*@`
- **Email addresses** in source — `[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}`.
  In this project one is expected: `FORMSUBMIT_ENDPOINT` in `index.html` carries the
  notification address by design. Confirm it is the intended one, confirm it appears
  **nowhere else**, and ask the user whether they are content for it to be public —
  a repo on GitHub is world-readable and the address will be scraped.
- **Internal identifiers**: real hostnames, internal IPs (`10.`, `192.168.`, `172.16-31.`),
  UNC paths (`\\server\share`), absolute local paths that leak a username
  (e.g. `C:\Users\<name>\`), and any real customer, staff or bank data. This repo is a
  training demo for a *fictional* PMO — real UOB data, logos or trademarks must not
  appear.
- **Files that should not ship**: `.env`, `.env.*`, `*.pem`, `*.key`, `*.pfx`, `*.p12`,
  `id_rsa*`, `*.keystore`, `credentials.json`, `secrets.*`, `*.sqlite`, `*.bak`,
  cloud CLI config (`.aws/`, `.azure/`, `.kube/config`), and anything large or
  third-party that is not redistributable (this repo already gitignores `*.pdf` course
  material and `Notes-*.txt` working notes — verify those rules still hold).

Useful commands:

```
git ls-files --cached --others --exclude-standard
git check-ignore -v <path>          # prove a file really is ignored
git log --all --name-only --pretty=format: | sort -u   # anything ever committed
```

Then:

- If you find a **live credential**: stop. Do not push. Report it, tell the user to
  rotate it, and only continue once it is removed from the working tree *and* from
  history if it was ever committed.
- If you find something **merely private but not dangerous** (an email, a local path):
  report it and ask before continuing.
- If the tree is clean: say so explicitly, listing what you checked, and continue.

Also confirm `.gitignore` covers the sensitive/large files you found. If it does not,
add entries and mention it.

## Step 2 — Upload the code to GitHub

1. If the target repo does not exist yet and `gh` is available, offer to create it and
   **ask whether it should be public or private**. Do not assume.
   `gh repo create OWNER/REPO --public --source . --remote origin` (or `--private`).
2. If `origin` is missing, `git remote add origin <url>`. If `origin` exists but points
   somewhere else than the target, show both and ask before changing it.
3. Stage deliberately, commit with a clear message, and push:
   `git push -u origin main`
4. Confirm the push landed: `git log --oneline -1 origin/main` and
   `git status -sb` should show no divergence.

If the remote already has commits you do not have, **do not force**. Fetch, show the
user the divergence, and ask how they want to reconcile it.

Commit messages end with:
`Co-Authored-By: Claude Opus 5 (1M context) <noreply@anthropic.com>`

## Step 3 — GitHub Pages via GitHub Actions

Check for `.github/workflows/*.yml` that deploys Pages. If one already exists, read it
and only fix what is actually wrong — do not rewrite a working workflow.

If none exists, create `.github/workflows/deploy-pages.yml` using the Actions-based
deployment (not the legacy `gh-pages` branch):

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
        with:
          enablement: true          # turns Pages on via the API if not already
      - uses: actions/upload-pages-artifact@v3
        with:
          path: .                   # adjust if the site is built into a subdir
      - id: deployment
        uses: actions/deploy-pages@v4
```

Adjust `path:` to the real output directory if the project has a build step. This
project is a single self-contained `index.html`, so `path: .` is correct.

Also ensure a `.nojekyll` file exists at the repo root — without it Pages runs Jekyll
and silently drops files and directories beginning with `_`.

Then:

- Push the workflow and watch the run: `gh run list --limit 3` /
  `gh run watch` if `gh` is available; otherwise give the user the Actions tab URL.
- The published URL is `https://OWNER.github.io/REPO/` (or `https://OWNER.github.io/`
  for a `OWNER.github.io` repo). Verify it responds once the run is green —
  first deployments can take a minute or two.
- If Pages is not enabled and `enablement: true` fails (it needs the repo's Actions
  permissions to allow it), tell the user to set
  **Settings → Pages → Build and deployment → Source: GitHub Actions**.

## Step 4 — Screenshot of the live site

Capture the published page and commit it as `docs/screenshot.png`, so the README can
show what the project looks like without the reader having to open it.

Do this only once Step 3 reports the site returning HTTP 200 — screenshotting a page
that is still deploying yields a 404 image.

**Preferred: the Playwright MCP tools** (`browser_navigate`, `browser_take_screenshot`).
The server is registered for this project in `.mcp.json`. If those tools are not in
your tool list, the server is not connected — say so rather than pretending to use it,
and fall back below. It needs Node.js on PATH, and a Claude Code restart after the
config is first added.

```
browser_navigate      → https://OWNER.github.io/REPO/
browser_take_screenshot → filename: docs/screenshot.png, fullPage: false
```

Wait for the page to settle before capturing. A viewport shot (`fullPage: false`) reads
better in a README than a long full-page strip; use `fullPage: true` only if the page
is short.

**Fallback when Playwright is unavailable** — headless Chrome or Edge, both of which
ship on Windows and need no install:

```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" --headless=new --disable-gpu `
  --screenshot="$PWD\docs\screenshot.png" --window-size=1440,900 `
  --virtual-time-budget=6000 --hide-scrollbars "https://OWNER.github.io/REPO/"
```

It must be `--headless=new`. Chrome removed the old headless mode in v132, and with a
bare `--headless` the browser exits 0 and writes **no file at all** — a silent no-op
that looks like success. Always check the file exists afterwards rather than trusting
the exit code.

`--virtual-time-budget` gives scripts time to render before the capture; without it you
can get a blank or half-drawn page. Swap in `msedge.exe` if Chrome is absent.

Note PowerShell renders a native command's stdout as a red `NativeCommandError` here —
that is cosmetic, not a failure. Judge by whether the PNG exists.

Either way, afterwards:

- Confirm the file exists and is a plausible size (a few hundred KB; a sub-5KB PNG
  usually means a blank or error page was captured).
- Open it and check it actually shows the app, not a 404 or an empty frame. Do not
  commit a screenshot you have not looked at.
- Re-capture whenever the UI changes, so the README never shows a stale interface.

## Step 5 — README

Read the existing `README.md` first if there is one, and edit rather than replace —
preserve anything the user wrote by hand.

It should cover, at a level appropriate to the project:

- Project title and a one-line description of what it is
- A **live demo link** to the Pages URL from Step 3
- The **screenshot from Step 4**, near the top, as
  `[![Screenshot](docs/screenshot.png)](<pages url>)` so the image itself links to the
  live site. Give it real alt text — never an empty `![]()`
- What it demonstrates / key features
- How to run it locally (for this project: open `index.html` — no build, no server)
- Notable constraints or caveats worth stating up front (for this project: no
  persistence by design, single file, no frameworks, no external resources)
- Tech stack, and licence/status if the user wants one

Keep it honest — do not claim features that are not implemented, and do not add
badges for CI that does not exist.

## Step 6 — Repo About + homepage link

Set the repo description and the homepage URL to the Pages link, and add topics.

With `gh`:

```
gh repo edit OWNER/REPO \
  --description "<one-line description>" \
  --homepage "https://OWNER.github.io/REPO/" \
  --add-topic <topic> --add-topic <topic>
```

With a token instead:

```
curl -X PATCH -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO \
  -d '{"description":"...","homepage":"https://OWNER.github.io/REPO/"}'
```

With neither, tell the user exactly where to click: the repo page → the ⚙ gear next to
**About** → Description, Website, Topics. Offer to paste the exact text for each field.

Confirm afterwards: `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`

## Step 7 — Report

Finish with a short summary:

- Secret scan result — what was checked, what was found, what was decided
- Commit pushed (SHA + message) and the repo URL
- Workflow run status and the live Pages URL (say if it is still deploying)
- Screenshot — captured with which tool, or why it was skipped
- What changed in the README
- About/homepage/topics — set, or the manual steps left for the user

State plainly anything you could not do and why. Do not report a step as done if it
was skipped or is still pending.
