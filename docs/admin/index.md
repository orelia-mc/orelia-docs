# 管理者ガイド

Oreliaのサーバー管理者コマンドは、動いているプラグインによって入口が分かれています。

| プラグイン | コマンド | 権限 | 内容 |
|---|---|---|---|
| orelia-core | `/oladmin` | `orelia.admin`(既定OP) | リロード・モンスター/ボス湧かせ・スポーンポイント・ダンジョン開放ブロック・NPC・アイテム配布など |
| orelia-debug(任意導入) | `/oladmin`(同じ入口に追加登録) | `orelia.admin`(既定OP) | テストプレイ・バランス調整支援(GUI強制表示・お金/経験値付与・設定編集など) |
| orelia-serverutil(任意導入) | `/suadmin` | `orelia.serverutil.admin`(既定OP) | ハブ転送・ワールドセットアップなど、RPG機能に依存しないサーバー運用機能 |

`orelia-debug`は元々`orelia-world`/`orelia-extra`という別プラグインを前提に「(要OreliaWorld)」のように書かれていた説明が残っていますが、現在はそれらの機能もすべて`orelia-core`に統合されているため、`orelia-core`さえ導入されていれば全コマンドが使えます。

## まず知っておくこと

- `/oladmin help [ページ]` — 現在導入されているプラグインが登録した全サブコマンドの一覧を確認できます。`/oladmin help <サブコマンド名>` と指定すると、そのサブコマンド1件だけの説明を表示します。
- `/oladmin reload` — 設定ファイルを再読み込みします。**このリロードで実際に変更された項目が一覧表示される**ので、設定ファイルを手で編集したあとの確認に使えます。
- 設定ファイル自体をゲーム内で確認・編集したい場合は、`orelia-debug`導入時に使える [`/oladmin config`](debug-tools.md#config) が便利です(YAMLの木構造をそのまま人間が読める形式で表示し、クリックで編集コマンドを自動入力できます)。

## 各ページ

- [orelia-core 管理コマンド](core-commands.md)
- [orelia-debug テストプレイ支援](debug-tools.md)
- [orelia-serverutil](serverutil.md)
- [コマンド一覧(早見表)](reference.md)
- [設定ファイルの書き方(config.yml)](config-guide.md)
- [設定ファイルの書き方(コンテンツ定義)](content-files.md)
- [ゲーム内ロジック](game-logic.md)
