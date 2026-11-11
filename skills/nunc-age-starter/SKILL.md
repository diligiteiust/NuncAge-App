---
name: nunc-age-starter
description: Set up a starter Nunc Age dashboard and example to-do lists so the app is not empty on first use. Use when the Nunc Age data folder is empty, the user is new to the app, or they ask for a sample or example dashboard.
---

# Nunc Age starter dashboard

This skill writes a small, generic starter dashboard into the Nunc Age data folder so the menu and dashboard have something to show. Load the `nunc-age` skill first and follow its file rules.

Example files live in [assets/](assets/). Copy them, then fill in dates relative to today. Do not leave the placeholder words in the CSV.

## When to write, and when to stop

- If the folder already has real lists (any `todos/*.csv` other than leftover starter files the user asked you to replace), do not overwrite them. Offer to add a new source, or stop and say what is already there.
- If the folder is missing, create `~/NuncAge` (or the path the user gave you), plus `todos/`, `dashboard/`, and `sections/`.
- If only a starter is present and the user wants a fresh one, it is fine to replace the starter source, not other sources.

## What to create

Use source id `starter` and display name `Starter` unless the user picks another name.

1. Register a row in `sources.csv` without removing other rows. Append `icon` when that column is not already there:

   ```
   source,name,notify_url,icon
   starter,Starter,,sf:checklist
   ```

   Leave `notify_url` blank unless this agent can receive a webhook, in which case follow `nunc-age` to add it.

2. Write `sections/starter.csv` from [assets/sections/starter.csv](assets/sections/starter.csv). These are SF Symbols and colors for Today, Work, Personal, Home, and Waiting. Do not put this file under `todos/`.

3. Write `todos/starter.csv` from [assets/todos/starter.csv](assets/todos/starter.csv), replacing the date tokens:
   - `TODAY` — today's date in the user's local time zone, `YYYY-MM-DD`
   - `TOMORROW` — the next calendar day
   - `IN_3_DAYS` — today plus 3 calendar days
   - `NEXT_MONDAY` — the next Monday (if today is Monday, use the following Monday)

   Keep the ids, sections, priorities, and icons. The icon column is the kind of work, and the same kind uses the same icon. Put today, overdue, and urgent rows first inside each section. The menu shows the first open rows of each section, and the app does not sort them. Rewrite the file when the order changes. You may drop or rewrite a row if it does not fit the user, but keep at least one item in Today, Work, Personal, Home, and Waiting so the dashboard shows how grouping works.

4. Write `dashboard/starter.md` from [assets/dashboard/starter.md](assets/dashboard/starter.md). You may shorten it, but keep the short "how this works" note so a first-time user knows that ticking an item notifies the agent. Keep it to a summary line and a small table.

5. Tell the user to open Nunc Age, confirm Settings points at this folder, and look at the menu and the dashboard. Mention that these are examples they can edit or delete, and that your AI agent should use a different `source` id for real lists (for example `planner`).

## Do not

- Do not invent due dates in the past just to demo "overdue" unless the user asks for that.
- Do not put personal, financial, or account data in the starter.
- Do not commit `ticks.csv` or `.nuncage/`.
