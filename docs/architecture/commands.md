# コマンド体系

## トップレベルコマンド

| コマンド | プラグイン | 権限 | デフォルト |
|---|---|---|---|
| `/ol` | orelia-core | なし | 全員 |
| `/oladmin` | orelia-core | `orelia.admin` | op |
| `/rpgworldadmin` | orelia-world | `orelia.world.admin` | op |
| `/rpgquest` | orelia-world | `orelia.quest` | 全員 |
| `/dialoguechoice` | orelia-world | `orelia.dialogue` | 全員（内部利用専用） |

`orelia-core` の権限ノードは `orelia.admin` の1つのみです。サブコマンド単位の権限は宣言されていません。

## `/ol` と `/oladmin` の設計思想

`OlCommandRegistry`（基底クラス）は `Map<String, CommandExecutor>` を保持し、`register(name, executor)` / `get(name)` / `getNames()` を提供します。2つの具象サブクラス：

- `PlayerCommandRegistry` — プレイヤー向け、ゲート無し
- `AdminCommandRegistry` — `/oladmin` の親権限でゲートされる

両方とも `ServicesManager` 経由で公開されるため、`orelia-world`（や将来の `orelia-extra`）は新しいトップレベルコマンドを作らず、同じ2つのエントリーポイントへサブコマンドを登録できます。

### `/ol` のディスパッチ（`OlRootCommand`）

引数なし → `registry.getNames()` の一覧を表示。`args[0]` を registry から検索し、見つかれば `label + " " + args[0]` とラベルを付け替えて残り引数（`Arrays.copyOfRange(args,1,...)`）を転送。見つからなければ `"Unknown subcommand: X"`。

### `/oladmin` のディスパッチ（`AdminCommand`）

**純粋な委譲ではなく**、いくつかのサブコマンドを直接ハンドリングし、未知の名前だけ `AdminCommandRegistry` へフォールバックします。

- `reload` → `plugin.reload()`
- `spawn <monsterId>` → `MonsterModule` 必須。`monsterModule.getSpawnService().spawn(id, player.getLocation())`
- `spawnboss <bossId>` → `BossModule` 必須。`bossModule.spawn(id, location)`
- `spawnpoint add <monsterId> [intervalSeconds=30] [maxAlive=3]` / `spawnpoint remove <id>` / `spawnpoint list` → `MonsterModule.getSpawnPointService()`
- それ以外 → `registry.get(args[0])` に転送。見つからなければ使用方法を表示

!!! info "AdminCommandRegistry への登録はゼロ"
    orelia-core内では `AdminCommandRegistry` へ登録されるサブコマンドは1つもありません。`/oladmin` の全アクションは `AdminCommand` のswitch文で直接処理されています。

### orelia-core が登録する `/ol` サブコマンド

| サブコマンド | 登録元 | 使い方 |
|---|---|---|
| `item` | `ItemModule` → `ItemCommand` | `/ol item give <player> <id> [amount]` |
| `job` | `JobModule` → `JobCommand` | `/ol job` / `/ol job list`（職業変更はNPC経由。このコマンドでは行わない） |
| `status` | `GuiModule` → `StatusCommand` | `/ol status`（ステータスGUIを開く） |

### orelia-world が持つ独自コマンド

`orelia-world` は `/ol`・`/oladmin` へサブコマンドを追加するのではなく、独自のトップレベルコマンドを`plugin.yml`で宣言しています。

| コマンド | 実装 | 使い方 |
|---|---|---|
| `/rpgworldadmin reload` | `WorldAdminCommand` | `OreliaWorldPlugin.reload()` を呼ぶ（現時点で唯一のサブコマンド） |
| `/rpgquest` | `QuestCommand` | `/rpgquest list`（進行中クエストと状態を表示）、`/rpgquest abandon <id>`（報酬なしで放棄） |
| `/dialoguechoice <index>` | `DialogueChoiceCommand` | チャット上のクリック可能な選択肢から内部的に呼ばれる。プレイヤーが直接打つことは想定しない |
