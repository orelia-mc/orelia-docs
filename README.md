<img src="https://orelia-mc.github.io/assets/logo_wide.jpg" />
<h1 align="center">Orelia Docs</h1>
<p align="center">Documentation of Orelia-MC</p>

## About

`orelia-docs` は Minecraft RPG プラグイン群 **Orelia** の [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 製ドキュメントサイトです。プラグイン本体のソースコードは含まず、隣接リポジトリ `orelia-core` / `orelia-world` を読んで書き起こした仕様書のみを収録しています。

公開サイト: https://orelia-mc.github.io/orelia-docs/

## Setup

```bash
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # http://127.0.0.1:8000 でライブプレビュー
mkdocs build --strict             # site/ にビルド(警告があれば失敗)
```

## Structure

- `docs/architecture/` — モジュールライフサイクル、Config システム、DB 層、プレイヤーデータ、コマンド体系など両プラグイン共通の設計
- `docs/core/` — `orelia-core` の各ゲームプレイモジュール(Item / Skill / Job / Status / Accessory / Monster / Boss / Effect / Economy / GUI)と公開 API (`rpg.api.*`)
- `docs/world/` — `orelia-world` の各コンテンツモジュール(Quest / NPC / Dialogue / Story / Dungeon / Region / CutScene / Event)

ナビゲーションは `mkdocs.yml` の `nav:` で明示的に定義されています。新しいページを追加した場合は必ずここにも追記してください。

`site/` はビルド成果物(gitignore 対象)です。`main` への push で CI が `mkdocs build --strict` を実行し、GitHub Pages にデプロイします。