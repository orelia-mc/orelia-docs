# orelia-debug 概要

`orelia-debug` は管理者向けのテストプレイ／デバッグ支援プラグインです。自身はゲームプレイ状態を一切所有せず、他のOreliaプラグインが公開する `rpg.api.*` / `rpg.world.api.*` / `rpg.extra.api.*` インターフェース（Bukkitの`ServicesManager`経由）に乗り入れるだけの薄い層です。本番サーバーへの導入は必須ではなく、開発・検証環境への導入を想定しています。

## 依存関係

- **orelia-core**（必須・`plugin.yml` `depend`） — `AdminCommandRegistry`、`ConfigManager`/`MessageManager`（いずれも`rpg.core.*`を再利用）に加えて、`DebugApi`/`GuiApi`/`EconomyApi`/`StatusApi`サービスを取得します。`onEnable`時にこれらのいずれかが見つからない場合はプラグインを無効化します（`AdminCommandRegistry`の欠落と`DebugApi`/`GuiApi`/`EconomyApi`/`StatusApi`いずれかの欠落は、それぞれ別のガードで検査されます）。
- **orelia-world**（任意・`softdepend`） — `WorldDebugApi`サービスを取得しますが、未導入時は`null`のまま起動を続行します。`null`の場合、`quest`サブコマンドおよび`config`/`confighelp`の`world`ターゲットは「OreliaWorldが導入されていません」という応答を返します。
- **orelia-extra**（任意・`softdepend`） — `ExtraDebugApi`サービスを取得しますが、未導入時は`null`のまま起動を続行します。`null`の場合、`gui auction|mail|ranking`および`config`/`confighelp`の`extra`ターゲットは「OreliaExtraが導入されていません」という応答を返します。
- **Vault**（`plugin.yml`の`softdepend`に記載）— orelia-core自身がVault経済プロバイダとして動作するため、orelia-debugからの直接依存はありません（`money`/`exp`はorelia-coreの`EconomyApi`/`StatusApi`を経由します）。

## コマンド登録の実際の形

`OreliaDebugPlugin#onEnable`は、orelia-coreの`AdminCommandRegistry`へサブコマンドを個別のフラットなエントリとして登録します。「`debug`」という中間サブコマンドは存在せず、各コマンドは直接 `/oladmin gui` 、`/oladmin money` のように呼び出します（`/oladmin` 自体はorelia-coreが所有し、`orelia.admin`権限・デフォルトop限定です。サブコマンド単位の権限ノードはorelia-core側にも存在しません）。

登録されるハンドラクラス（すべて`rpg.debug.command`パッケージ、`CommandExecutor`/`TabCompleter`）:

| サブコマンド | ハンドラクラス | 使用するAPI |
|---|---|---|
| `gui` | `GuiDebugCommand` | `GuiApi`（必須）、`ExtraDebugApi`（任意） |
| `money` | `MoneyDebugCommand` | `EconomyApi`（必須） |
| `config` | `ConfigDebugCommand` | `DebugApi`（必須）、`WorldDebugApi`/`ExtraDebugApi`（任意） |
| `confighelp` | `ConfigHelpDebugCommand` | `DebugApi`（必須）、`WorldDebugApi`/`ExtraDebugApi`（任意） |
| `quest` | `QuestDebugCommand` | `WorldDebugApi`（必須・未導入時は即エラー） |
| `exp` | `ExpDebugCommand` | `StatusApi`（必須） |
| `manual` | `ManualCommand`（内部で`DebugManual`） | なし（静的テキストのページ送り表示） |

!!! info "`npc` サブコマンドはorelia-debugには存在しない"
    `DebugManual`（`/oladmin manual`のページ内容）には`oladmin npc create|move|remove|list`の説明が含まれますが、これは**orelia-world本体が提供するコマンド**の案内であり、orelia-debug自身が登録・処理するサブコマンドではありません（`OreliaDebugPlugin#onEnable`にも`npc`の`register`呼び出しはありません）。README.mdも同様に「NPCの一覧・設置・移動・削除は`orelia-debug`ではなく`orelia-world`本体の`/oladmin npc`コマンドで行ってください」と明記しています。

## `/oladmin` サブコマンド一覧

すべて`orelia.admin`権限（デフォルトop）で保護されます。プレイヤーを指定する引数は省略すると原則コマンド送信者自身が対象になります（コンソールから省略した場合は`command.player-only`エラー）。

| コマンド | 引数 | 動作 | 備考 |
|---|---|---|---|
| `/oladmin gui <status\|equipment\|skill\|job\|shop\|warehouse\|auction\|mail\|ranking> [player]` | 画面名、対象プレイヤー省略可 | 指定プレイヤー（省略時は自分）に対して指定GUIを通常のプレイ導線を経由せず直接開く | `shop`は`guiApi.openShop(target, List.of())`＝在庫なしの状態で開く。`auction`/`mail`/`ranking`は`ExtraDebugApi`が`null`なら`gui.extra-not-installed`を返す |
| `/oladmin money <give\|set\|take> [player] <amount>` | 種別、対象プレイヤー省略可、金額（double） | `EconomyApi.deposit`/`setBalance`/`withdraw`を呼び所持金を操作 | 金額がdoubleとしてパースできない場合`money.invalid-amount`。`take`は`withdraw`が`false`を返すと`money.take-failed`（残高不足） |
| `/oladmin config <core\|world\|extra> list` | 対象プラグイン | 対象の登録済み設定ファイル名一覧を表示 | `world`/`extra`は対応する`*DebugApi`が`null`なら未導入エラー |
| `/oladmin config <core\|world\|extra> get <file> <path>` | 対象、ファイル名、YAMLドット区切りパス | 現在値を表示 | 値が存在しない場合`config.value-not-found` |
| `/oladmin config <core\|world\|extra> set <file> <path> <value>` | 対象、ファイル名、パス、値（文字列） | 値を書き換えて**即座にファイルへ保存** | `true`/`false`→boolean、数値として解釈できればlong/double、それ以外は文字列として自動判定（`DebugApi`実装側の変換ロジック。orelia-debug自体はraw文字列をそのまま渡すのみ） |
| `/oladmin config <core\|world\|extra> save <file>` | 対象、ファイル名 | 設定ファイルを手動保存 | |
| `/oladmin confighelp <core\|world\|extra> <file>` | 対象、ファイル名 | そのファイルの全設定キー一覧を表示 | キーが0件の場合`config.keys-empty` |
| `/oladmin quest complete [player] <questId>` | 対象プレイヤー省略可、クエストID | `WorldDebugApi.forceCompleteQuestObjectives`で受注中クエストの全目標を強制的に達成状態（`AWAITING_REPORT`）へ遷移させる | 報酬付与は行わない。対象プレイヤーが`/ol quest`からNPCへ報告して受け取る必要がある。OreliaWorld未導入時は引数チェックより先に`gui.world-not-installed`を返す |
| `/oladmin exp give [player] <amount>` | 対象プレイヤー省略可、経験値（long） | `StatusApi.addExperience`で経験値を直接付与 | 金額がlongとしてパースできない場合`exp.invalid-amount` |
| `/oladmin manual [page]` | ページ番号（省略時1） | README.mdの内容をゲーム内でページ送り表示（1ページ6件×2行） | ページ番号が数値でなければ1ページ目にフォールバック |

## 設定・メッセージ

orelia-debug自身は`config.yml`のようなゲームプレイ設定ファイルを持たず、`ConfigManager`に登録するのは`messages.yml`（ユーザー向け文言、`prefix`/`command.*`/`usage.*`/`gui.*`/`money.*`/`config.*`/`quest.*`/`exp.*`キー）のみです。全ての応答文言は`MessageManager`経由でこのファイルから送信され、コマンドコード側に`ChatColor`＋リテラル文字列をハードコードしない規約になっています（`messages.yml`冒頭のコメントに明記）。

## リロード

`OreliaDebugPlugin#reload()`は`configManager.reloadAll()`を呼びますが、これを呼び出す専用のadminサブコマンド（例えば`/oladmin debugreload`）は登録されていません。他プラグイン（orelia-core/orelia-world/orelia-extra）と異なり、orelia-debugには`/oladmin`経由のリロードコマンドが存在しない点に注意してください。
