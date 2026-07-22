# Config システム

`orelia-core` と `orelia-world` はそれぞれ独立した `ConfigManager` インスタンスを持ちますが、実装パターンは共通です。

## `ConfigManager`

- `register(String fileName)` — `computeIfAbsent` で管理。`plugin.getResource(name) != null` であれば `plugin.saveResource(name, false)` でjar同梱のデフォルトを（上書きせず）コピーし、`new ConfigFile(logger, dataFolder, fileName)` を構築。ファイルが**既に存在していた**場合（＝新規コピーではなく既存ユーザーファイル）は、続けて [config-version 自動マイグレーション](#config-version-自動マイグレーション) を実行する。
- `get(String fileName)` — 未登録なら `IllegalStateException("Config file not registered: ...")`。
- `reload(String fileName)` / `reloadAll()` — 登録済み全ファイルに対し `ConfigFile::reload` を呼ぶ。
- `getRegisteredFileNames()` — 登録済み全ファイル名（デバッグAPI `DebugApi` 向け）。

`ConfigManager` はモジュール固有のキーには一切踏み込みません。各モジュールが自分のYAMLの中身を解釈します。

## config-version 自動マイグレーション

`ConfigManager.register()` が既存ファイルを検出すると `ConfigMigrator.migrate(logger, file, bundledText)` を呼びます（`rpg.core.config.ConfigMigrator`、パッケージプライベート）。

- 各ファイルの先頭に置く整数キー `config-version` で世代を管理します（未指定は `0` 扱い）。jar同梱のデフォルトの方がユーザーファイルより新しい場合のみマイグレーションが走ります。新しいトップレベルキーを追加したときだけ番号を上げる運用で、値だけの変更（デフォルト値の調整など）ではインクリメントしません。
- マイグレーションは **`YamlConfiguration` でロード→再保存する方式を意図的に取りません**（コメントが全て失われるため）。代わりに、デフォルトファイルのテキストを空行区切りのトップレベルブロック単位でパースし、ユーザーファイルに無いキーのブロックだけをコメントごと生テキストのまま末尾に追記します。
- ユーザーファイルにあるがデフォルトにはもう存在しないキーは、`WARNING` ログ（`"Config key 'X' in <file> is no longer used by this version - you can remove it."`）を出すのみで、**自動削除はしません**。
- ブロック抽出は「空行でセクションを区切り、キー行の直上の `#` コメントをそのセクションに含める」という、このリポジトリの `config.yml` の書式を前提としています。この形に沿わないセクションは自動追記の対象外（従来通り手動追記が必要）。

`orelia-core` の `config.yml` は現在 `config-version: 3` です。

## `ConfigFile`

`YamlConfiguration` を `File` でラップします。

- `load()` — 親ディレクトリがなければ作成し、`YamlConfiguration.loadConfiguration(file)`。
- `reload()` — `load()` と同じ。
- `save()` — `configuration.save(file)`。`IOException` は `SEVERE` ログ。
- `get()` — 生の `YamlConfiguration` を返す。
- `getFile()` — バッキングファイルを返す。

!!! warning "スレッドセーフではない"
    リロードはメインスレッドで `/oladmin reload` / `/rpgworldadmin reload` からのみ発生する前提です。

## リロードコマンド

| プラグイン | コマンド | 動作 |
|---|---|---|
| orelia-core | `/oladmin reload` | `configManager.reloadAll()` → `moduleManager.reloadAll()` |
| orelia-world | `/rpgworldadmin reload` | `configManager.reloadAll()` → `moduleManager.reloadAll()` |

## orelia-core `config.yml` の実際のキー

```yaml
config-version: 3

database:
  type: SQLITE            # SQLITE または MYSQL
  sqlite:
    file: orelia.db
  mysql:
    host: localhost
    port: 3306
    database: orelia
    username: orelia
    password: ""
    use-ssl: false

status:
  regen:
    hp-percent-per-tick: 0.5
    sp-percent-per-tick: 1.0
    period-ticks: 100
  leveling:
    exp-per-level: 100
    max-level: 100
  growth:
    HP:  { base: 100, per-level: 10 }
    SP:  { base: 50,  per-level: 5 }
    ATK: { base: 5,   per-level: 1 }
    # ... StatType（HP, SP, ATK, DEF, AGI, DEX, INT, CRT, CRT_DMG, SPD）ごとに1ブロック

economy:
  starting-balance: 100.0

monster:
  disable-vanilla-hostile-spawning: true
```

## `messages.yml`

`messages.yml` は `rpg.core.message.MessageManager`（各プラグインが自分の `messages.yml` に対して1つ持つ）経由で、コマンド/リスナー/サービスに散らばっていたハードコードされた `ChatColor + "..."` 文字列を置き換えるために使われます。

- キーはドット区切りのパス（例: `job.changed`、`admin.reloaded`、`command.unknown-subcommand`）。
- テンプレート中の `{name}` トークンは `format`/`send` の可変長引数（`name, value, name, value, ...` の順）で位置ベースに置換されます。
- 未定義キーは例外を投げず、キー自体を `??key??` で囲んで返します（タイポがゲーム内表示で即座に分かる）。
- `send(sender, key, ...)` は `getPrefix()` を前置し、`sendRaw(sender, key, ...)` は複数行の一覧やGUIタイトルなど prefix 不要な場面向け。
- 色付けは [`ColorUtil`](commands.md#カスタムカラーコードcolorutil) の `&` コード（vanillaレガシー・`&#RRGGBB` ヘックス・`&%<char>` カスタムコードいずれも）に対応します。

## orelia-world `config.yml` の実際のキー

```yaml
quest:
  objective-check-period-ticks: 40  # 2秒。QuestModule の周期目標チェッカーが使用
```

## モジュール別の設定ファイル一覧

| プラグイン | ファイル |
|---|---|
| orelia-core | `config.yml`, `items.yml`, `skills.yml`, `jobs.yml`, `accessories.yml`, `monsters.yml`, `bosses.yml`, `effects.yml`, `gui.yml`, `messages.yml` |
| orelia-world | `config.yml`, `quests.yml`, `npc.yml`, `dungeons.yml`, `dialogues.yml`, `story.yml`, `regions.yml`, `cutscenes.yml`, `events.yml` |
