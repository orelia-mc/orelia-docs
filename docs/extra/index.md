# orelia-extra 概要

`orelia-extra` は Paper 1.21.x (Java 21) 向けの**後発MMORPG機能層**プラグインです。`orelia-core`（`depend: [OreliaCore]`）が有効化された後にのみロードされ、`softdepend: [OreliaWorld, Vault]` を持ちます。

- `orelia-core`（別リポジトリ） — 戦闘/プレイヤー/ステータス基盤。必須依存
- `orelia-world`（別リポジトリ） — クエスト/NPC/ストーリーのコンテンツ層。ソフト依存（`AchievementModule` の `COMPLETE_QUEST` 条件のみ）
- **orelia-extra**（本体） — Party, Guild, Trade, Mail, Auction, Housing, Pet, Mount, Ranking, Achievement

## 統合ルール

`orelia-extra` は `orelia-core` / `orelia-world` の内部ゲームプレイモジュールクラス（`rpg.status.*`, `rpg.item.*`, `rpg.quest.*` など）へ**絶対に**直接アクセスしません。統合面は次の2つのみです。

1. `rpg.api.*`（`StatusApi`, `SkillApi` など）/ `rpg.world.api.*`（`QuestApi`）、いずれも `ServicesManager` 経由
2. 汎用の `rpg.core.*` / `rpg.database.*` インフラ（`ConfigManager`, `SchedulerService`, `PlayerDataManager`, `DatabaseManager`, `SchemaOwner`）

お金のやり取り（オークション決済、住居/ペット/マウント購入など）は Vault の `Economy` を直接使います。専用の `EconomyApi` は存在しません。

## Composition Root（`OreliaExtraPlugin.onEnable()`）

1. `ServicesManager` から orelia-core の `PlayerDataManager` を取得（無ければプラグインを disable してハードフェイル）
2. `ServicesManager` から orelia-core の `PlayerCommandRegistry` / `AdminCommandRegistry` を取得（無ければ同様にハードフェイル）
3. 自前の `ConfigManager`（`config.yml` を登録） / `SchedulerService` を構築
4. `ExtraAdminCommand` を `AdminCommandRegistry` へ `"extrareload"` として登録（`/oladmin extrareload`）
5. モジュールを**おおむねアルファベット順**に登録（登録順＝有効化順、逆順で無効化）：

    ```
    PartyModule → GuildModule → TradeModule → MailModule → AuctionModule → HousingModule → PetModule → MountModule → RankingModule → AchievementModule
    ```

6. `moduleManager.enableAll()`

Ranking と Achievement が最後なのは、両方とも自分ではデータを所有せず、他モジュール（や orelia-core / orelia-world）が持つ状態を読むだけの立場だからです。

## モジュール一覧

| モジュール | 概要 | 消費する API |
|---|---|---|
| [Party](party.md) | インメモリのパーティ機能（create/invite/accept/leave/kick/disband） | なし |
| [Guild](guild.md) | DB永続化されたギルドとleader/officer/memberロール | なし |
| [Trade](trade.md) | confirm/confirm ハンドシェイクによる2人間アイテム取引（インメモリ） | なし |
| [Mail](mail.md) | アイテム添付・GUI受信箱付きのDB永続化メールボックス | なし |
| [Auction](auction.md) | 期限付き出品のプレイヤー主導オークションハウス | Vault Economy |
| [Housing](housing.md) | 設定駆動の購入可能な住居プロットと`/ol house home`テレポート | Vault Economy |
| [Pet](pet.md) | 設定駆動の追従ペット（unlock/summon/dismiss） | Vault Economy |
| [Mount](mount.md) | 設定駆動の騎乗マウント（unlock/summon/dismiss） | Vault Economy |
| [Ranking](ranking.md) | レベルランキングGUI。データはすべて`StatusApi`から | StatusApi |
| [Achievement](achievement.md) | 設定駆動の実績（レベル/クエスト/所持金条件）、報酬はスキルポイント | StatusApi, SkillApi, Vault Economy, QuestApi（ソフト） |

## Config ファイル

`config.yml`（Partyの`party.max-size`など横断設定）, `achievements.yml`, `housing.yml`, `mounts.yml`, `pets.yml`。詳細は[Configシステム](../architecture/config.md)。

## リロード

`/oladmin extrareload` で全モジュールの設定ファイルを再読込します（orelia-core の `AdminCommandRegistry` に登録されているため `/oladmin` のサブコマンドとして動作し、`/oladmin reload`・`orelia-world`の`/rpgworldadmin reload`とは独立しています）。

## コマンド

| コマンド | 用途 |
|---|---|
| `/ol party <create\|invite\|accept\|leave\|kick\|disband\|list>` | パーティ管理 |
| `/ol guild <create\|invite\|accept\|leave\|kick\|promote\|demote\|disband\|info>` | ギルド管理 |
| `/ol trade <player>\|accept\|add\|remove <index>\|confirm\|cancel\|view` | 取引 |
| `/ol mail [unread]` | メールボックスGUIを開く／未読件数表示 |
| `/ol auction [list\|sell <price>\|collect]` | オークションGUIを開く／出品／未売却回収 |
| `/ol house [list\|buy <plotId>\|home]` | 住居プロットの一覧／購入／帰宅 |
| `/ol pet [list\|buy <id>\|summon [id]\|dismiss]` | ペットの一覧／購入／召喚／解除 |
| `/ol mount [list\|buy <id>\|summon [id]\|dismiss]` | マウントの一覧／購入／召喚／解除 |
| `/ol ranking` | レベルランキングGUIを開く |
| `/ol achievement` | 実績の達成状況一覧 |
| `/oladmin extrareload` | 全モジュールの設定リロード（`orelia.admin`、op） |
