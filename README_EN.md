<img src="https://orelia-mc.github.io/assets/logo_wide.jpg" />
<h1 align="center">Orelia Docs</h1>
<p align="center">Documentation of Orelia-MC</p>

## About

`orelia-docs` is the [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)-based documentation site for the Minecraft RPG plugin suite **Orelia**. It contains no plugin source code — only specs written by reading the sibling repos `orelia-core` / `orelia-world` / `orelia-extra` / `orelia-debug` / `orelia-serverutil`.

Live site: https://orelia-mc.github.io/orelia-docs/

## Setup

```bash
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # live preview at http://127.0.0.1:8000
mkdocs build --strict             # build to site/ (fails on any warning)
```

## Structure

- `docs/architecture/` — design shared by all three plugins: module lifecycle, the Config system, the DB layer, player data, and the command system
- `docs/core/` — each gameplay module of `orelia-core` (Item / Skill / Job / Status / Accessory / Monster / Boss / Effect / Economy / GUI) and the public API (`rpg.api.*`)
- `docs/world/` — each content module of `orelia-world` (Quest / NPC / Dialogue / Story / Dungeon / Region / CutScene / Event)
- `docs/extra/` — each feature module of `orelia-extra` (Party / Guild / Trade / Mail / Auction / Housing / Pet / Mount / Ranking / Achievement)
- `docs/debug/` — overview and command reference for `orelia-debug`, the admin-only testplay/debug tooling plugin
- `docs/serverutil/` — overview for `orelia-serverutil`, the server-operations/UX plugin that's independent of the RPG suite
- `docs/verification/` — post-install checklist and per-feature in-game verification steps (Job / Skill / Item / Status / Accessory / Gathering / Monster / Boss / Economy / GUI / Effect)

Navigation is explicitly defined in `mkdocs.yml`'s `nav:`. Any new page must be added there too.

`site/` is build output (gitignored). CI runs `mkdocs build --strict` and deploys to GitHub Pages on every push to `main`.
