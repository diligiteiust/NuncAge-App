# Nunc Age file format

Nunc Age 1.0 reads a folder of plain files. Any agent — a script, a bot, Claude Code, or something else — can feed it. Nothing in this format is specific to one agent.

The app never edits agent files. The only files it writes are `ticks.csv` and a private retry queue under `.nuncage/`.

## Folder

The default folder is `~/NuncAge`. The path is configurable in Settings.

```
NuncAge/
  todos/<source>.csv
  dashboard/<source>.md
  sources.csv
  ticks.csv          # written only by the app; do not commit it
  .nuncage/          # written only by the app; do not commit it
```

`<source>` is the file name without the extension. `todos/planner.csv` is source `planner`. One file per agent. Only that agent writes it, and it rewrites the whole file each time.

## todos/<source>.csv

RFC 4180 CSV with a header row. Quote a field that contains a comma, a quote, or a newline. A quote inside a quoted field is `""`.

Unquoted fields are trimmed. Quote a field when the leading or trailing spaces matter. A quote that is never closed is treated as closed at the end of the file.

| Column | Required | Meaning |
| --- | --- | --- |
| `id` | yes | Unique within this file. The last row wins if an id is repeated. |
| `title` | yes | One line. Inline Markdown links are shown as links. |
| `section` | no | Group name on the dashboard. Empty sections are labeled General. |
| `due` | no | `YYYY-MM-DD`. Anything else is ignored. |
| `priority` | no | `1` high, `2` normal, `3` low. Blank or any other value is `2`. |
| `status` | no | `open` or `done`. Only the exact word `done` counts as done. |
| `done_at` | no | When the agent marked it done. ISO 8601, see below. |
| `note` | no | Optional second line under the title. May contain newlines if quoted. |
| `link` | no | Optional `http` or `https` URL. The app opens it from the row. |

Unknown columns are ignored, so the format can grow. When you rewrite the file, keep any extra columns you still need. The header names are lowercase. `id` has to be present.

`done_at` accepts:

- `YYYY-MM-DD` (the start of that day in the user's local time zone)
- `YYYY-MM-DDThh:mm:ssZ`
- fractional seconds, such as `YYYY-MM-DDThh:mm:ss.sssZ`
- a numeric offset, `+hh:mm`, `+hhmm`, or the same with `-`

The app writes timestamps as UTC, for example `2026-11-10T15:04:05Z`.

### Example

```csv
id,title,section,due,priority,status,done_at,note,link
p1,"Read the [format spec](https://example.com/format)",Reading,2026-11-09,1,open,,,"https://example.com/format"
p2,Draft the weekly note,Writing,2026-11-10,2,open,,"Bring the comma, and keep going",
```

## How done is decided

The app shows an item as done when either of these is true:

- The latest tick for that source and id is `done`.
- The file says `status` `done`, and there is no newer `undone` tick.

Ticks are ordered by `at`. If two ticks have the same time, the later line in `ticks.csv` wins. A later `undone` wins over an earlier tick.

An `undone` tick is newer than the file when its time is greater than or equal to `done_at`. If the file says `done` but `done_at` is missing, an `undone` tick wins. If the agent rewrites the file as `done` with a `done_at` later than the untick, the file wins again.

Write `done_at` whenever you set `status` to `done`. The menu does not list completed items. The dashboard shows a completed item when it falls in the range chosen in Settings (this week by default). A completed item with no timestamp is shown only when that range is always.

## ticks.csv

The app appends this file. Agents do not write it.

```csv
at,source,id,action
2026-11-10T15:10:00Z,mail,m2,done
```

`action` is `done` or `undone`. The header must be exactly `at,source,id,action`.

## Notify

`sources.csv` names each agent and, optionally, where to tell it about ticks:

```csv
source,name,notify_url
planner,Planner,https://example.com/hooks/planner?token=example-token
mail,Mail,
```

`notify_url` is optional. It contains its own secret. The app never logs the full URL.

When the user ticks or unticks, the app appends `ticks.csv` and, if that source has a usable `http` or `https` notify URL, POSTs JSON:

```json
{"action":"done","at":"2026-11-10T15:10:00Z","id":"m2","source":"mail","title":"Archive newsletters"}
```

`Content-Type` is `application/json`. `action` is `done` or `undone`. Delivery is at-least-once: the app keeps a queue under `.nuncage/` and retries with backoff if the POST fails. Treat `(source, id, at)` as the event. On a notify, update your CSV (`status`, and `done_at` when the action is `done`) and rewrite the file.

## dashboard/<source>.md

Optional. The dashboard shows this Markdown above that source's to-dos, grouped by `section` in file order.

The renderer understands:

- Headings (`#` through `######`, with a space after the hashes)
- Paragraphs
- Bullet lists (`-`, `*`) and numbered lists (`1.` or `1)`)
- Tables (a header row, a `| --- |` delimiter, then body rows)
- A line of `---`, `***`, or `___` is a break and is not shown

Fenced code is shown as plain text. List items are one line each. Inline Markdown (links, emphasis) is rendered inside those blocks.

## What the user sees

Row order is priority order. Put the most important items first: today, overdue, and urgent at the top. When the order should change, rewrite the file. The app does not sort the menu by due date or priority.

The menu shows the first N open rows of each `todos/<source>.csv`, in that file's row order, under the source's name from `sources.csv`. N is a Settings value and starts at 3. Ticking an item takes it off the menu, and the next open row from that file takes its place. The menu-bar count is the number of open items that are overdue or due today.

The dashboard shows every open item from every source, and completed items for the chosen range. Due dates and priority stay on those rows. Items are grouped by `section` in file order. The default range is the current Monday–Sunday week, local time. Settings can keep completed items for the last 7 days, for 30 days, or always.

Sources follow `sources.csv`, then any other source name alphabetically.
