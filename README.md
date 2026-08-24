![orelia-docs](https://socialify.git.ci/orelia-mc/orelia-docs/image?description=1&language=1&name=1&owner=1&theme=Dark)

## About

`orelia-docs` は Minecraft RPG プラグイン **Orelia** の [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 製ドキュメントサイトです。**プレイヤー向けの遊び方ガイドと、サーバー管理者向けのコマンドリファレンス** で構成されており、内部実装の仕様書ではありません。プラグイン本体のソースコードは含まず、隣接リポジトリ `orelia-core`(統合済みの単一プラグイン本体) / `orelia-debug`(テストプレイ支援) / `orelia-serverutil`(サーバー運用) のコマンド仕様・ゲーム内挙動を読んで書き起こしています。

公開サイト: https://orelia-mc.github.io/orelia-docs/

## Setup

```bash
pip install -r requirements.txt   # mkdocs-material>=9.7
mkdocs serve                      # http://127.0.0.1:8000 でライブプレビュー
mkdocs build --strict             # site/ にビルド(警告があれば失敗)
```

## Structure

- `docs/play/` — プレイヤー向けガイド。レベル/ステータス/職業、戦闘・武器スキル、装備・アイテム、クエスト・NPC、ダンジョン、パーティー・ギルド・フレンド、チャット、経済(ショップ・トレード・オークション・メール)、実績・ランキング・称号、住居・ペット・乗り物、採取、コマンド一覧
- `docs/admin/` — 管理者向けガイド。`orelia-core`組み込みの`/oladmin`コマンド、任意導入の`orelia-debug`(テストプレイ支援)コマンド、`orelia-serverutil`の`/suadmin`・`/hub`、コマンド一覧

ナビゲーションは `mkdocs.yml` の `nav:` で明示的に定義されています。新しいページを追加した場合は必ずここにも追記してください。

`site/` はビルド成果物(gitignore 対象)です。`main` への push で CI が `mkdocs build --strict` を実行し、GitHub Pages にデプロイします。
