# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working from this repository.

## What this is

`orelia-docs` is the MkDocs Material documentation site for the Orelia Minecraft RPG plugin suite. It is a **player and server-admin facing guide**, not developer/architecture documentation — how to play (leveling, jobs, quests, dungeons, parties/guilds, trading, ...) and how to run admin commands (what to type, what it does), not how the plugin is implemented internally (no class names, no data models, no SQL schemas). This repo contains no plugin source code itself.

The suite it documents is `../orelia-core` (the single merged plugin — formerly split across orelia-core/orelia-world/orelia-extra, now one jar), plus two optional companion plugins: `../orelia-debug` (admin-only testplay/debug tooling) and `../orelia-serverutil` (RPG-independent server-ops/UX).

## Commands

```
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # live-reload preview at http://127.0.0.1:8000
mkdocs build --strict             # build to site/ — fails on any warning (broken nav entry, bad link, etc.)
```

`site/` is build output (gitignored) — never edit it directly. CI runs `mkdocs build --strict` on every push to `main` and deploys `site/` to GitHub Pages via `.github/workflows/deploy.yml` (`actions/upload-pages-artifact` + `actions/deploy-pages`).

## Structure

Navigation is defined in `mkdocs.yml` (`nav:`), not inferred from the filesystem — any new page must be added there explicitly or it won't appear in the site.

- `docs/play/` — the player guide, one page per system a player actually interacts with (growth/status/jobs, combat/weapon skills, items/equipment, quest/NPC, dungeon, party/guild/friend, chat, economy, achievements/ranking/titles, housing/pet/mount, gathering), plus a player command quick-reference table.
- `docs/admin/` — the admin guide, one page per plugin's admin command surface (`orelia-core`'s built-in `/oladmin` commands, `orelia-debug`'s testplay tooling — also under `/oladmin`, `orelia-serverutil`'s separate `/suadmin`/`/hub`), plus an admin command quick-reference table. Each command gets its own heading with syntax, required permission (usually just "`/oladmin` needs `orelia.admin`, default OP" stated once, not repeated per command), effect, and an example.

## Keeping docs accurate

This documentation is written by reading the actual command classes and `messages.yml` in the sibling repos (`../orelia-core`, `../orelia-debug`, `../orelia-serverutil`) — the exact `case`/`switch` branches a command dispatches on, not by paraphrasing an old registration description string (those can drift out of date, e.g. a command's one-line description in its `AdminCommandRegistry.register(...)` call not being updated when a new subcommand like `config view` was added later — always verify against the command class's actual `onCommand` logic).

When updating a page after upstream code changes:

- Re-read the actual command source in the sibling repo rather than editing prose speculatively.
- Keep the level of detail practical (command syntax, GUI button layout, what happens, a real example) — not implementation detail (class/field names, service method signatures, SQL schemas, damage formulas). If you find yourself describing a Java class, it belongs in that repo's own `CLAUDE.md`/README, not here.
- `orelia-debug`'s bundled command descriptions still say things like "(要OreliaWorld)" ("requires OreliaWorld") from before the 3-plugin merge — that's stale (everything is bundled into `orelia-core` now), don't propagate it into this site's prose.
- If you find a documented behavior that no longer matches the source, fix the doc — the sibling repos are the source of truth, this repo is derived from them.

## Committing changes

When committing, also update README.md and README_EN.md accordingly.
