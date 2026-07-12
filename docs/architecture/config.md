# Config システム

`orelia-core` と `orelia-world` はそれぞれ独立した `ConfigManager` インスタンスを持ちますが、実装パターンは共通です。

## `ConfigManager`

- `register(String fileName)` — `computeIfAbsent` で管理。`plugin.getResource(name) != null` であれば `plugin.saveResource(name, false)` でjar同梱のデフォルトを（上書きせず）コピーし、`new ConfigFile(logger, dataFolder, fileName)` を構築。
- `get(String fileName)` — 未登録なら `IllegalStateException("Config file not registered: ...")`。
- `reload(String fileName)` / `reloadAll()` — 登録済み全ファイルに対し `ConfigFile::reload` を呼ぶ。

`ConfigManager` はモジュール固有のキーには一切踏み込みません。各モジュールが自分のYAMLの中身を解釈します。

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

`messages.yml` も存在しますが、コメントにある通り**現時点ではどのコードからも参照されていません**（将来のメッセージ一元化のために予約されたファイル）。

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
