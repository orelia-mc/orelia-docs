<img src="https://orelia-mc.github.io/assets/logo_wide.jpg" />
<h1 align="center">Orelia Docs</h1>
<p align="center">Documentation of Orelia-MC</p>

## About

`orelia-docs` is the [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)-based documentation site for the Minecraft RPG plugin **Orelia**. It's a **player how-to-play guide and a server-admin command reference**, not internal-implementation documentation. It contains no plugin source code — only command specs and in-game behavior written by reading the sibling repos `orelia-core` (the single merged plugin) / `orelia-debug` (testplay tooling) / `orelia-serverutil` (server operations).

Live site: https://orelia-mc.github.io/orelia-docs/

## Setup

```bash
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # live preview at http://127.0.0.1:8000
mkdocs build --strict             # build to site/ (fails on any warning)
```

## Structure

- `docs/play/` — the player guide: growth/status/jobs, combat/weapon skills, items/equipment, quest/NPC, dungeons, party/guild/friend, chat, economy (shops/trade/auction/mail), achievements/ranking/titles, housing/pet/mount, gathering, and a player command reference
- `docs/admin/` — the admin guide: `orelia-core`'s built-in `/oladmin` commands, `orelia-debug`'s (optional) testplay tooling, `orelia-serverutil`'s `/suadmin`/`/hub`, and an admin command reference

Navigation is explicitly defined in `mkdocs.yml`'s `nav:`. Any new page must be added there too.

`site/` is build output (gitignored). CI runs `mkdocs build --strict` and deploys to GitHub Pages on every push to `main`.
