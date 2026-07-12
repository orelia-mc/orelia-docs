# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`orelia-docs` is the MkDocs Material documentation site for the Orelia Minecraft RPG plugin suite. It documents two sibling repositories that live alongside this one (`../orelia-core`, `../orelia-world`) — this repo contains no plugin source code itself, only hand-written specs derived from reading that source.

## Commands

```
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # live-reload preview at http://127.0.0.1:8000
mkdocs build --strict             # build to site/ — fails on any warning (broken nav entry, bad link, etc.)
```

`site/` is build output (gitignored) — never edit it directly. CI runs `mkdocs build --strict` on every push to `main` and deploys `site/` to GitHub Pages via `.github/workflows/deploy.yml` (`actions/upload-pages-artifact` + `actions/deploy-pages`).

## Structure

Navigation is defined in `mkdocs.yml` (`nav:`), not inferred from the filesystem — any new page must be added there explicitly or it won't appear in the site.

- `docs/architecture/` — cross-cutting design shared by both plugins: module lifecycle (`RpgModule`/`WorldModule`, registration order = dependency order, enable forward / disable reverse), the `ConfigManager`/`ConfigFile` config system, the shared SQLite/MySQL database layer (`Repository`/`SchemaOwner`), the `PlayerData`/`PlayerDataComponent` per-player state system, and the `/ol` · `/oladmin` · `/rpgworldadmin` command dispatchers.
- `docs/core/` — one page per `orelia-core` gameplay module (Item, Skill, Job, Status, Accessory, Monster, Boss, Effect, Economy, GUI), plus `core/api.md` documenting every `rpg.api.*` interface method — this is the **only** integration surface `orelia-world`/`orelia-extra` are allowed to call into `orelia-core` through.
- `docs/world/` — one page per `orelia-world` content module (Quest, NPC, Dialogue, Story, Dungeon, Region, CutScene, Event), each noting which `rpg.api.*` interfaces it consumes.

## Keeping docs accurate

This documentation was written by reading `orelia-core`/`orelia-world` source directly (model fields, YAML config keys, service method behavior, formulas), not by paraphrasing existing docs or guessing from class names. When updating a page after upstream code changes:

- Re-read the actual source/config in the sibling repo rather than editing prose speculatively.
- Preserve the existing per-module structure (domain model → real YAML example → services/formulas → commands → cross-module API usage) so pages stay consistent with each other.
- If you find a documented behavior that no longer matches the source, fix the doc — the sibling repos are the source of truth, this repo is derived from them.
- A few known integration gaps are intentionally documented as such (e.g. `orelia-world`'s dungeon-clear not yet triggering quest `CLEAR_DUNGEON` objectives, `EventScheduleService` multipliers not yet applied to reward grants) — don't silently "fix" these in the docs by describing wiring that doesn't exist in the code.
