---
name: nunc-age
description: Connect to Nunc Age, a macOS menu-bar TODO and dashboard app that reads a shared folder of CSV and Markdown files. Use when installing or configuring Nunc Age, writing or updating the user's to-do lists or dashboard, registering a notify URL, or handling a tick (done or undone) from the app.
---

# Nunc Age

Nunc Age is a macOS 14+ menu-bar app. Agents write plain files into a shared folder (default `~/NuncAge`). The app never edits those files. When the user ticks or unticks an item, the app appends `ticks.csv` and, if you registered a notify URL, POSTs JSON to you.

The full file contract is in [references/FORMAT.md](references/FORMAT.md). Follow it exactly. This skill is the working recipe.

If the data folder is empty and the user wants a starting dashboard, also load the `nunc-age-starter` skill.

## 1. Find the folder

1. The default path is `~/NuncAge` on the Mac that runs the app.
2. If the folder is missing, ask before creating it, unless the user already asked you to set Nunc Age up.
3. If Settings in the app points somewhere else, use that path. Do not guess a second folder.
4. Create `todos/`, `dashboard/`, `sections/`, and `icons/` as needed. `sections/` is not inside `todos/`.

## 2. Register yourself

`sources.csv` lists every agent. Columns: `source,name,notify_url,icon`. The `icon` column is optional. A file without it still works. When you add it, append it at the end.

- `source` is a short stable id and must match your filenames: `todos/<source>.csv`, `dashboard/<source>.md`, and `sections/<source>.csv`. Use lowercase letters, numbers, and hyphens (for example `planner`).
- `name` is the display name in the app.
- `notify_url` is optional. Add an `http` or `https` URL only if you can receive a POST at that address. The URL may contain a secret; never log or reprint the full URL.
- `icon` is optional. It is a path relative to the data folder, such as `icons/planner.png`, or an SF Symbol such as `sf:checklist`. Rules are in [references/FORMAT.md](references/FORMAT.md).

If `sources.csv` already exists, keep every existing row. Add or update only your own row. If the file does not exist, create it with the header and your row.

One agent, one source. Do not reuse another agent's `source` id.

## 3. Write your lists

Write only your own files. Never edit another source's CSV or Markdown, and never write `ticks.csv` or anything under `.nuncage/`. You may add a file under the shared `icons/` folder. Do not change an icon file you did not add.

Rewrite the whole file each time you change it (do not append a partial row). Keep any extra columns you still need.

`todos/<source>.csv` is RFC 4180 CSV with this header:

```
id,title,section,due,priority,status,done_at,note,link,icon
```

- `id` is required and unique within this file. The last row wins if an id is repeated.
- `title` is required, one line. Inline Markdown links are shown as links on the dashboard. The menu shows the link's label.
- `section` groups rows on the dashboard. Empty sections show as General.
- `due` is `YYYY-MM-DD`, or leave it blank. Any other value is ignored.
- `priority` is `1` (high), `2` (normal), or `3` (low). Blank or anything else is `2`.
- `status` is `open` or `done`. Only the exact word `done` counts as done.
- `done_at` is required whenever you set `status` to `done`. Use ISO 8601 UTC, such as `2026-11-10T15:04:05Z`, or `YYYY-MM-DD`.
- `note` is an optional second line under the title.
- `link` is an optional `http` or `https` URL.
- `icon` is optional. Append it at the end. A file without the column still works. Use the same value rules as a section icon. On a row, this icon replaces the section icon. The section icon stays on the section header.

Row order is priority order. Put the most important items first inside each section: today, overdue, and urgent at the top of that section. The menu shows the first open rows of each section (the user chooses 3, 5, 10, or all; the default is 5). One agent can own many sections, so the cap is per section, not per file. The dashboard shows every open item, and completed items for as long as the user keeps them (this week by default). When the order should change, rewrite the file. Do not rely on the app to sort by due date or priority.

Quote a field that contains a comma, a quote, or a newline. A quote inside a quoted field is `""`.

`sections/<source>.csv` is optional. Header: `section,icon,color`. `section` matches the name in your to-do file. `color` is `#RRGGBB` and tints an SF Symbol and the section card. Do not put this file under `todos/`.

`icons/` is shared. Reuse a picture another agent already put there, such as `icons/github.png` or `icons/quickbooks.png`. Add a file you need. Do not delete or replace someone else's picture.

`dashboard/<source>.md` is optional Markdown shown inside your source card, above the sections. Headings, paragraphs, bold, italics, lists, tables, block quotes, images, inline links, and card rows work. A card row is a `::: cards` block: each `###` heading is a card sitting side by side. An image is `![alt](icons/chart.png)`, relative to the data folder, with the same path rules as an icon.

## 4. Stay in sync with ticks

The app shows an item as done when the latest tick for that source and id is `done`, or when your file says `done` and there is no newer `undone` tick. Details are in [references/FORMAT.md](references/FORMAT.md).

When you are notified of a tick, update that row in your CSV (`status`, and `done_at` when the action is `done`) and rewrite the file. Treat `(source, id, at)` as the event; delivery is at-least-once, so ignore a duplicate.

You do not read `ticks.csv` to discover ticks. That file stays on the Mac and is not committed. Use `notify_url` (or the user telling you) instead.

### Notify payload

`POST`, `Content-Type: application/json`:

```json
{"action":"done","at":"2026-11-10T15:10:00Z","id":"m2","source":"mail","title":"Archive newsletters"}
```

`action` is `done` or `undone`.

## 5. Make it look good

People recognize a picture faster than a line of text. Give every section an icon, and give an item an icon when the kind of work is obvious. Use one icon for one kind of work. Do not invent a different icon for every row.

For a section, use the project's real app icon when you can find it. Copy a PNG out of the app bundle or the repo into `icons/`, and point the section or the source at that file. When you cannot find a picture, use an SF Symbol:

- finances: `sf:dollarsign.circle.fill`
- music: `sf:music.note`
- games: `sf:gamecontroller.fill`

Other sections follow the same idea: a fitting symbol, or a real logo in `icons/`.

For an item, pick the icon for the kind of work:

- coding: `sf:chevron.left.forwardslash.chevron.right`
- GitHub, a pull request, or a repo: `icons/github.png` when that file is already there, otherwise `sf:arrow.triangle.branch`
- QuickBooks or bookkeeping: `icons/quickbooks.png` when that file is already there, otherwise `sf:book.closed.fill`
- email or a message: `sf:envelope.fill`
- a call: `sf:phone.fill`
- a payment or a bill: `sf:creditcard.fill`
- taxes or forms: `sf:doc.text.fill`
- a release or an upload: `sf:arrow.up.circle.fill`
- design or an icon: `sf:paintbrush.fill`
- music: `sf:music.note`
- a meeting or a date: `sf:calendar`
- waiting on someone: `sf:hourglass`

Keep the dashboard note short enough to scan. No walls of text. One summary line, a small table, and an optional image still work. When a few facts belong next to each other, such as usage, a personal bank, an LLC bank, and Stripe, put them in one card row instead of stacking full-width tables.

A line that trims to exactly `::: cards` opens a row. A line that trims to exactly `:::` closes it. With no closing line, the row runs to the end of the file. Each `###` heading inside the row starts a card. The heading is the title. A trailing `{icon}` uses a section icon (`sf:name` or a path such as `icons/bank.png`). An invalid icon is dropped from the title. Text before the first `###` is a card with no title. The body of a card is normal Markdown (paragraphs, lists, tables, quotes, images). Do not nest a row. A `::: cards` line inside a row is plain text, and a fenced code block wins over the row. Use 2–4 cards per row and about 3 table rows per card. Several rows are fine. A column of amounts (`+`, `−`, `-`, or `$`) is right-aligned. The details are in [references/FORMAT.md](references/FORMAT.md).

## 6. Git

If the folder is a git checkout, the app can run `git pull --ff-only` on its own. `ticks.csv` and `.nuncage/` must not be committed; the app adds them to `.git/info/exclude`. When you commit and push list changes, leave those paths untracked.

## Using Grok Bot

Grok Bot is a hosted assistant. It runs on its own cloud computer, not on the user's Mac. It installs `nunc-age` and `nunc-age-starter` from GitHub, then writes the same files in the user's data folder through the user's approved access to that Mac. Follow every rule in this skill: one source, rewrite your own files, never write `ticks.csv` or `.nuncage/`, and do not log a `notify_url`.

## What not to do

- Do not invent a second data folder.
- Do not write `ticks.csv`.
- Do not delete or rewrite another agent's files.
- Do not log a `notify_url`.
- Do not mark an item done in the CSV without writing `done_at`.
- Do not put a section file under `todos/`. It belongs in `sections/`.
- Do not point an icon at a path outside the data folder.
