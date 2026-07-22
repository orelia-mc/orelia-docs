# コマンド体系

## トップレベルコマンド

| コマンド | プラグイン | 権限 | デフォルト |
|---|---|---|---|
| `/ol` | orelia-core | なし | 全員 |
| `/oladmin` | orelia-core | `orelia.admin` | op |

`orelia-core` の権限ノードは `orelia.admin` の1つのみです。サブコマンド単位の権限は宣言されていません。

!!! warning "orelia-world は独自のトップレベルコマンドを持たない"
    どちらの `plugin.yml` にも `commands:` セクションはありません。`/rpgworldadmin`・`/rpgquest`・`/dialoguechoice` という独立トップレベルコマンドは存在せず、`orelia-world` の全コマンドは `/ol`・`/oladmin` のサブコマンドとして登録されています（下記「`orelia-world` が登録するサブコマンド」参照）。加えて `quest` は [`CommandAliasUtil`](#動的コマンドエイリアスcommandaliasutil) 経由で `/quest` としても実行できます。

## `/ol` と `/oladmin` の設計思想

`OlCommandRegistry`（基底クラス）は `Map<String, Entry>` を保持します。`Entry(name, executor, description, usage)` は `/ol help`・`/oladmin help` の表示に使うメタデータを実行体と一緒に持ち、`register(name, executor, description, usage)` / `get(name)` / `getEntry(name)` / `getNames()` / `getEntries()` を提供します。2つの具象サブクラス：

- `PlayerCommandRegistry` — プレイヤー向け、ゲート無し
- `AdminCommandRegistry` — `/oladmin` の親権限でゲートされる

両方とも `ServicesManager` 経由で公開されるため、`orelia-world`（や将来の `orelia-extra`）は新しいトップレベルコマンドを作らず、同じ2つのエントリーポイントへサブコマンドを登録できます。

## `/ol help` / `/oladmin help`

`args[0]` が省略されているか `help` の場合、両ディスパッチャーは `CommandHelpUtil.sendHelp(sender, label, entries, page)` に委譲します。`entries` は登録済みの `OlCommandRegistry.Entry` 一覧（`/oladmin` の場合は `reload`/`spawn`/`spawnboss`/`spawnpoint` のハードコードされたビルトインエントリーに `AdminCommandRegistry` の登録内容を追加したもの）で、`/ol help [page]` / `/oladmin help [page]` の2つ目の引数（数値でなければ1ページ目扱い）でページを指定できます。表示は共通の [`Pagination`](#ページングpagination) に委譲され、各行は `/<rootLabel> <usage>&%7 - <description>` の形式です。

## 共通コマンドユーティリティ（`rpg.core.command`）

コマンドUX改善（#38, #39）で、`/ol`・`/oladmin` 系コマンド全体が使う以下の共通部品が新設されました。

### お金のフォーマット（`rpg.util.MoneyFormat`）

`MoneyFormat.format(double)` はGUI/スコアボード/チャット/プレースホルダーで金額を表示する共通フォーマッタです。1000未満はそのまま（整数でなければ小数第1位まで）、以降は桁ごとに `k`（千）→`m`（百万）→`b`（十億）→`t`（一兆）表記に切り替わります（例: `1500` → `1.5k`、`2000000` → `2m`、`3_000_000_000` → `3b`、`4_000_000_000_000` → `4t`）。負数には `-` 符号を残します。`ShopGuiScreen` の価格lore・購入成功メッセージが `MoneyFormat` に統一されました（詳細は [Economy モジュール](../core/economy.md#表示フォーマット)）。

### ページング（`rpg.core.command.Pagination`）

一覧が1画面に収まらない `/ol`・`/oladmin` コマンド（help、その他今後の一覧系コマンド）向けの共通ページング機構です。`Pagination.send(sender, titleTemplate, lines, pageSize, page, baseCommand[, emptyText])` は区切り線・タイトル（`{page}`/`{total}` トークン置換）・クリック可能な「« 前へ」「次へ »」ナビゲーション行（`ColorUtil.componentWithCommand` で `<baseCommand> <pageNumber>` を実行）を描画します。ページ範囲外の指定は自動的にクランプされます。`CommandHelpUtil` はこれに委譲する形にリファクタされました。

### 共通タブ補完（`rpg.core.command.TabCompletions`）

`TabCompletions.matching(options, prefix)`（大文字小文字を無視した前方一致フィルタ）と `TabCompletions.onlinePlayerNames(prefix)`（オンラインプレイヤー名の前方一致）が公開ヘルパーとして新設され、`ItemCommand`・`JobCommand` など各サブコマンドの `onTabComplete` から共通利用されています。

### 引数省略時の自己解決（`rpg.core.command.CommandArgs`）

`CommandArgs.resolvePlayerName(sender, args, index)` は「対象プレイヤー引数が省略されたら実行者自身」を解決する共通ヘルパーです（コンソールから引数無しで実行した場合は `null`）。

### 動的コマンドエイリアス（`CommandAliasUtil`）

`CommandAliasUtil.registerAlias(plugin, name, executor, description, usage)` は `Server#getCommandMap()`（Bukkitの公開API）経由で、`plugin.yml` に宣言せずにトップレベルコマンドを実行時登録するユーティリティです。既に `PlayerCommandRegistry`/`AdminCommandRegistry` に登録済みの `CommandExecutor`/`TabCompleter` インスタンスをそのまま再利用するため、`/ol <name>` と `/<name>` は完全に同じ実行体・状態を共有します（`CommandMap#register` は名前衝突時に `fallbackPrefix` で名前空間化するだけで失敗しないため、エイリアス登録は常に安全）。これが「動的コマンドエイリアス基盤」で、現在2箇所で使われています。

- `orelia-core`: `GuiModule` が `/ol status` を登録した `StatusCommand` インスタンスを `/status` としてもエイリアス登録
- `orelia-world`: `QuestModule` が `/ol quest` を登録した `QuestCommand` インスタンスを `/quest` としてもエイリアス登録（`quest.command.QuestCommand` は変更なし、モジュール側の登録コードのみ追加）— **`orelia-world` のクエスト固有ドキュメントは `docs/world/quest.md` を参照。**

### カスタムカラーコード（`ColorUtil`）

`ColorUtil` は `&`コード文字列を `§`コードへ変換します。従来のバニラレガシーコード・`&#RRGGBB` ヘックスに加え、`&%<char>` 形式のカスタムカラーコードが `CUSTOM_COLORS` マップ経由でサポートされています（`0`–`9`/`a`–`h` に用途別の色を割り当て済み、未定義文字はそのままリテラル表示されタイポとして気づける）。`Pagination`・`CommandHelpUtil` を含む全てのコマンド出力・`messages.yml` のテキストがこの拡張込みで色付けされます。`componentWithCommand`/`componentWithUrl` はクリックでコマンド実行/URLオープンする `Component` を組み立てるヘルパーで、`Pagination` のページ送りリンクなどに使われています。

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
