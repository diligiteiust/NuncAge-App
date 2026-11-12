# Nunc Age file format

Nunc Age reads a folder of plain files. Any agent — a script, a bot, Claude Code, or something else — can feed it. Nothing in this format is specific to one agent.

The app never edits agent files. The only files it writes are `ticks.csv` and a private retry queue under `.nuncage/`.

## Folder

The default folder is `~/NuncAge`. The path is configurable in Settings.

```
NuncAge/
  todos/<source>.csv
  dashboard/<source>.md
  sections/<source>.csv   # optional icons and colors for sections
  icons/                   # optional shared pictures
  sources.csv
  ticks.csv          # written only by the app; do not commit it
  .nuncage/          # written only by the app; do not commit it
```

`<source>` is the file name without the extension. `todos/planner.csv` is source `planner`. One file per agent. Only that agent writes it, and it rewrites the whole file each time.

`sections/` is not inside `todos/`. A CSV under `todos/` is a source. A CSV under `sections/` only describes that source's sections.

`icons/` is shared. Several sources can point at the same file, for example `icons/github.png` or `icons/quickbooks.png`. Do not delete another agent's file there.

## todos/<source>.csv

RFC 4180 CSV with a header row. Quote a field that contains a comma, a quote, or a newline. A quote inside a quoted field is `""`.

Unquoted fields are trimmed. Quote a field when the leading or trailing spaces matter. A quote that is never closed is treated as closed at the end of the file.

| Column | Required | Meaning |
| --- | --- | --- |
| `id` | yes | Unique within this file. The last row wins if an id is repeated. |
| `title` | yes | One line. Inline Markdown links are shown as links on the dashboard. |
| `section` | no | Group name on the dashboard and in the menu. Empty sections are labeled General. |
| `due` | no | `YYYY-MM-DD`. Anything else is ignored. |
| `priority` | no | `1` high, `2` normal, `3` low. Blank or any other value is `2`. |
| `status` | no | `open` or `done`. Only the exact word `done` counts as done. |
| `done_at` | no | When the agent marked it done. ISO 8601, see below. |
| `note` | no | Optional second line under the title. May contain newlines if quoted. |
| `link` | no | Optional `http` or `https` URL. The app opens it from the row. |
| `icon` | no | Optional icon for this item. Same rules as a section icon, below. |
| `options` | no | Optional choices for a question item, separated by `|`, such as `Ship today|Wait for review|Cancel`. Empty parts are ignored. |
| `allow_other` | no | Optional. `true`, `yes`, or `1` adds an Other… choice with a free-text field. Anything else, including a blank, is false. |

Unknown columns are ignored, so the format can grow. When you rewrite the file, keep any extra columns you still need. The header names are lowercase. `id` has to be present. A file with no `icon`, `options`, or `allow_other` column still works. Append those optional columns at the end so older files still parse. A missing or invalid icon is left blank. The app does not warn about it.

On a row, the item icon is shown in place of the section icon. The section icon stays on the section header.

`done_at` accepts:

- `YYYY-MM-DD` (the start of that day in the user's local time zone)
- `YYYY-MM-DDThh:mm:ssZ`
- fractional seconds, such as `YYYY-MM-DDThh:mm:ss.sssZ`
- a numeric offset, `+hh:mm`, `+hhmm`, or the same with `-`

The app writes timestamps as UTC, for example `2026-11-10T15:04:05Z`.

### Example

```csv
id,title,section,due,priority,status,done_at,note,link,icon,options,allow_other
p1,"Read the [format spec](https://example.com/format)",Reading,2026-11-09,1,open,,,"https://example.com/format",sf:book.fill,,
p2,Draft the weekly note,Writing,2026-11-10,2,open,,"Bring the comma, and keep going",,sf:pencil,,
q1,How should we ship 2.5?,Writing,,2,open,,Pick one or type your own,,sf:questionmark.circle,"Ship today|Wait for review|Cancel",true
```

See `samples/todos/` for a full folder.

## How done is decided

The app shows an item as done when either of these is true:

- The latest tick for that source and id is `done`.
- The file says `status` `done`, and there is no newer `undone` tick.

Ticks are ordered by `at`. If two ticks have the same time, the later line in `ticks.csv` wins. A later `undone` wins over an earlier tick.

An `undone` tick is newer than the file when its time is greater than or equal to `done_at`. If the file says `done` but `done_at` is missing, an `undone` tick wins. If the agent rewrites the file as `done` with a `done_at` later than the untick, the file wins again.

Write `done_at` whenever you set `status` to `done`. The menu does not list completed items. The dashboard shows a completed item when it falls in the range chosen in Settings (this week by default). A completed item with no timestamp is shown only when that range is always.

An item with `options` or `allow_other` is a question. The checkbox stays off until the user picks a choice or types Other text. Once a question is ticked, the answer is final: the checkbox and the pop-up lock, and the item cannot be unticked. Ordinary items can still be ticked and unticked. Answering is a premium feature, like ticking. Free can still read the question and its options.

## ticks.csv

The app appends this file. Agents do not write it.

```csv
at,source,id,action,answer,answer_type
2026-11-10T15:10:00Z,mail,m2,done,,
2026-11-10T15:12:00Z,planner,q1,done,Ship today,option
```

`action` is `done` or `undone`. `answer` and `answer_type` are filled when the user answers a question item. `answer_type` is `option` when they picked a listed choice, or `other` when they typed free text. They are empty on ordinary ticks. The usual header is `at,source,id,action,answer,answer_type`. Older files that still use `at,source,id,action` are read, and they stay in that form until a question is answered.

## Notify

`sources.csv` names each agent and, optionally, where to tell it about ticks. `icon` is optional. Files that do not have that column still work.

```csv
source,name,notify_url,icon
planner,Planner,https://example.com/hooks/planner?token=example-token,sf:checklist
mail,Mail,,sf:envelope.fill
```

`notify_url` is optional. It contains its own secret. The app never logs the full URL.

When the user ticks or unticks, the app appends `ticks.csv` and, if that source has a usable `http` or `https` notify URL, POSTs JSON:

```json
{"action":"done","at":"2026-11-10T15:10:00Z","id":"m2","source":"mail","title":"Archive newsletters"}
```

When the tick is an answer to a question item, the body also has `answer` and `answer_type`:

```json
{"action":"done","answer":"Ship today","answer_type":"option","at":"2026-11-10T15:12:00Z","id":"q1","source":"planner","title":"How should we ship 2.5?"}
```

`Content-Type` is `application/json`. `action` is `done` or `undone`. `answer_type` is `option` or `other`. Ordinary items omit `answer` and `answer_type`. Delivery is at-least-once: the app keeps a queue under `.nuncage/` and retries with backoff if the POST fails. Treat `(source, id, at)` as the event. On a notify, update your CSV (`status`, and `done_at` when the action is `done`) and rewrite the file. For a question, also read `answer` and `answer_type`; that answer is final.

## Icons

An icon value is either:

- a path relative to the data folder, for a PNG, JPEG, HEIC, or ICNS file, such as `icons/intellego.png`
- an SF Symbol, written `sf:` plus the symbol name, such as `sf:dollarsign.circle.fill`

The path must stay inside the data folder. A path that starts with `/` or `~`, or that contains `..`, is ignored. A symlink that points outside the folder is ignored. A missing file, a file that is not one of those image types, or an unknown SF Symbol is ignored. The app shows no icon and does not warn.

Use the same picture for the same kind of work. `icons/` is the shared place for brand marks such as `icons/github.png` and `icons/quickbooks.png`.

## sections/<source>.csv

Optional. One file per source, named like the source id: `sections/planner.csv` for source `planner`. It is not a to-do list.

```csv
section,icon,color
Reading,sf:book.fill,#1D4ED8
Admin,icons/receipts.png,#0F766E
```

| Column | Required | Meaning |
| --- | --- | --- |
| `section` | yes | The same section name used in `todos/<source>.csv`. The last row wins if a name is repeated. An empty name styles the General group. |
| `icon` | no | Icon for the section header. Same rules as above. |
| `color` | no | `#RRGGBB`. Tints an SF Symbol and the section card on the dashboard. Anything else is ignored. |

Unknown columns are ignored. A missing file is fine. A file that cannot be read, or that has no `section` column, is reported with the other file warnings. A bad icon or a bad color on an otherwise valid row is ignored.

## dashboard/<source>.md

Optional. The dashboard shows this Markdown inside that source's card, above the section cards. Items are grouped by `section` in file order.

The renderer understands:

- Headings (`#` through `######`, with a space after the hashes)
- Paragraphs, including `**bold**` and `*italics*`
- Bullet lists (`-`, `*`) and numbered lists (`1.` or `1)`)
- Tables (a header row, a `| --- |` delimiter, then body rows), drawn as a grid. A column whose body cells all start with `+`, `−`, `-`, or `$` is right-aligned.
- Money amounts are colored, in table cells and in other dashboard text. A leading minus is red (`−$39.88`, `-$10.00`, `-€5`). The minus can be `-` or `−`. A leading `+` is green (`+$37.57`, `+$4.99`), and after a sign the currency symbol is optional. Money without a sign counts as positive and is green too: a currency symbol before the digits (`$1,789`, `$27.07`, `€12`) or a currency code after them (`1,234.50 USD`). A plain number (`2.4`, `3 open`), a percentage (`9.6%`), a date (`2026-11-12`), a range (`1.3–2.5 s`, `$10–$20`), an amount in parentheses (`($130.26)`), and a trailing plus (`Lv 20+`) stay the normal color. Write a minus for money going out. Bold and other inline formatting still apply.
- Block quotes (`>`), drawn as a callout
- Images, `![alt](relative/path.png)` or `![alt](<relative/path with spaces.png>)`. The path uses the same rules as an icon and is relative to the data folder. The picture is limited to a modest width. A path that escapes the folder is not loaded.
- A card row, drawn side by side. A line that trims to exactly `::: cards` opens the row. A line that trims to exactly `:::` closes it. If the closing line is missing, the row runs to the end of the file. Inside the row, each `###` heading starts a card. The heading text is the title. An optional `{icon}` at the end of the title uses the same values as a section icon (`sf:name` or a relative image path in the data folder). An invalid icon is ignored and left off the title. Text before the first `###` is a card with no title. The card body uses the same blocks as the rest of the file: paragraphs, lists, tables, quotes, and images. A row does not nest. A `::: cards` line inside a row is plain text. Fenced code wins: a `::: cards` or `:::` line inside a fenced code block does not open or close a row, and a `###` line inside a fence does not start a card. The app lays cards out three to a row, all the same width and height, on a gray background. A fourth card starts a new row. Use about 3 table rows in a card. Several rows are fine.
- A line of `---`, `***`, or `___` is a break and is not shown

```
::: cards
### Grok Bot usage {sf:gauge.with.dots.needle.33percent}
**9.6%** of the weekly limit used

Resets **Wed, Nov 18**

### Personal checking {sf:building.columns.fill}
**$1,234.56** available

| Date | Description | Amount |
| --- | --- | --- |
| Nov 10 | Grocery store | −$42.10 |
:::
```

Fenced code is shown as plain text. List items are one line each. Inline Markdown (links, bold, italics) is rendered inside those blocks.

Keep the note short: one summary line, a small table of the numbers that matter, and an optional image. Put side-by-side facts in a card row instead of stacking full-width tables.

## What the user sees

Row order is priority order. Put the most important items first: today, overdue, and urgent at the top of each section. When the order should change, rewrite the file. The app does not sort the menu by due date or priority.

The menu is a normal menu. It grows with its contents. Each source is a disabled header, with its icon when it has one. Under that, each section is a header, with its icon when it has one. The menu shows the first N open items of each section, in file order. N is 3, 5, 10, or all of them. The default is 5. Ticking an item takes it off that section's list, and the next open row from the same section takes its place. The menu-bar count is the number of open items that are overdue or due today.

The dashboard shows every open item from every source, and completed items for the chosen range. Each source is a card. Each section is a card with its icon, its color, and a count of open and done items. Section cards sit three to a row. Due dates and priority stay on the rows. An overdue date is orange. A date of today uses the accent. Items are grouped by `section` in file order. The default range is the current Monday–Sunday week, local time. Settings can keep completed items for the last 7 days, for 30 days, or always.

Sources follow `sources.csv`, then any other source name alphabetically.
