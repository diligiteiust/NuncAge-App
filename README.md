# Nunc Age

Nunc Age is a macOS menu-bar TODO list and dashboard. You tick items in the app; your agent keeps the lists in a folder of plain CSV and Markdown files.

This public repository is for [issues](https://github.com/diligiteiust/NuncAge-App/issues), documentation, and the agent skills. The app source is private.

## Install the skills

There are two skills:

- **nunc-age** — how an agent talks to the app (the folder, the CSV and Markdown files, and tick notifications)
- **nunc-age-starter** — a sample dashboard and a few common lists, so the app is not empty on first launch

### From the app

In Nunc Age 1.1 or later, open Settings and use **Install for agents on this Mac**. That copies both skills into the skill folders of the agents it finds (Cursor, Claude Code, Codex, and the shared `~/.agents/skills` folder). Use **Save a zip** if you need to upload a skill to Claude on the web.

### From this repository

```sh
npx skills add diligiteiust/NuncAge-App -g
```

That installs the skills for your user account, in every supported agent the installer knows about. You can also clone this repository and copy `skills/nunc-age` and `skills/nunc-age-starter` into your agent's skills folder yourself.

## File format

The contract the app reads is [skills/nunc-age/references/FORMAT.md](skills/nunc-age/references/FORMAT.md). The default data folder is `~/NuncAge`.

## Support

Please [open an issue](https://github.com/diligiteiust/NuncAge-App/issues) for bugs, questions, and feature requests.
