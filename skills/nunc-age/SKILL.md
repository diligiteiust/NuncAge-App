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
4. Create `todos/` and `dashboard/` as needed.

## 2. Register yourself

`sources.csv` lists every agent. Columns: `source,name,notify_url`.

- `source` is a short stable id and must match your filenames: `todos/<source>.csv` and `dashboard/<source>.md`. Use lowercase letters, numbers, and hyphens (for example `planner`).
- `name` is the display name in the app.
- `notify_url` is optional. Add an `http` or `https` URL only if you can receive a POST at that address. The URL may contain a secret; never log or reprint the full URL.

If `sources.csv` already exists, keep every existing row. Add or update only your own row. If the file does not exist, create it with the header and your row.

One agent, one source. Do not reuse another agent's `source` id.

## 3. Write your lists

Write only your own files. Never edit another source's CSV or Markdown, and never write `ticks.csv` or anything under `.nuncage/`.

Rewrite the whole file each time you change it (do not append a partial row). Keep any extra columns you still need.

`todos/<source>.csv` is RFC 4180 CSV with this header:

```
id,title,section,due,priority,status,done_at,note,link
```

- `id` is required and unique within this file. The last row wins if an id is repeated.
- `title` is required, one line. Inline Markdown links are shown as links.
- `section` groups rows on the dashboard. Empty sections show as General.
- `due` is `YYYY-MM-DD`, or leave it blank. Any other value is ignored.
- `priority` is `1` (high), `2` (normal), or `3` (low). Blank or anything else is `2`.
- `status` is `open` or `done`. Only the exact word `done` counts as done.
- `done_at` is required whenever you set `status` to `done`. Use ISO 8601 UTC, such as `2026-11-10T15:04:05Z`, or `YYYY-MM-DD`.
- `note` is an optional second line under the title.
- `link` is an optional `http` or `https` URL.

Quote a field that contains a comma, a quote, or a newline. A quote inside a quoted field is `""`.

`dashboard/<source>.md` is optional Markdown shown above your to-dos. Headings, paragraphs, lists, tables, and inline links work. Keep it short.

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

## 5. Git

If the folder is a git checkout, the app can run `git pull --ff-only` on its own. `ticks.csv` and `.nuncage/` must not be committed; the app adds them to `.git/info/exclude`. When you commit and push list changes, leave those paths untracked.

## What not to do

- Do not invent a second data folder.
- Do not write `ticks.csv`.
- Do not delete or rewrite another agent's files.
- Do not log a `notify_url`.
- Do not mark an item done in the CSV without writing `done_at`.
