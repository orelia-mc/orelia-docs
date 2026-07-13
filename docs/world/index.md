# orelia-world 概要

`orelia-world` は Paper 1.21.x (Java 21) 向けの**コンテンツ層**プラグインです。`orelia-core`（`depend: [OreliaCore]`）が有効化された後にのみロードされ、`softdepend: [Vault]` を持ちます。

- `orelia-core`（別リポジトリ） — 戦闘/プレイヤー/ステータス基盤。必須依存
- **orelia-world**（本体） — Quest, NPC, Dialogue, Story, Dungeon, Region, CutScene, Event
- [`orelia-extra`](../extra/index.md)（別リポジトリ） — 後発MMORPG機能。`QuestApi` 経由でこのプラグインにソフト依存

## 統合ルール

`orelia-world` は `orelia-core` の内部ゲームプレイモジュールクラス（`rpg.status.*`, `rpg.item.*` など）へ**絶対に**直接アクセスしません。統合面は次の2つのみです。

1. `rpg.api.*`（`StatusApi`, `ItemApi`, `CombatApi`, `SkillApi`, `AccessoryApi`, `GuiApi`, `EffectApi` など、`ServicesManager` 経由）
2. 汎用の `rpg.core.*` / `rpg.database.*` インフラ（`ConfigManager`, `SchedulerService`, `PlayerDataManager`, `PlayerDataComponent`, `DatabaseManager`, `SchemaOwner`）

お金のやり取り（クエスト報酬、NPCショップなど）は Vault の `Economy` を直接使います。専用の `EconomyApi` は存在しません。

## Composition Root（`OreliaWorldPlugin.onEnable()`）

1. `ServicesManager` から orelia-core の `PlayerDataManager` を取得（無ければプラグインを disable してハードフェイル）
2. 自前の `ConfigManager` / `SchedulerService` を構築
3. `WorldAdminCommand` を `/rpgworldadmin` にバインド
4. モジュールを**依存順**に登録（登録順＝有効化順、逆順で無効化）：

    ```
    RegionModule → DialogueModule → StoryModule → EventModule → CutSceneModule → DungeonModule → QuestModule → NpcModule
    ```

5. `moduleManager.enableAll()`

`NpcModule` が最後なのは、クエスト受付NPCが `QuestModule` の登録済み状態に依存するためです。

## モジュール一覧

| モジュール | 概要 | 消費する `rpg.api.*` |
|---|---|---|
| [Quest](quest.md) | クエスト定義と受注→進行→報告→報酬の状態機械 | StatusApi, ItemApi, AccessoryApi, SkillApi, CombatApi |
| [NPC](npc.md) | 静的NPC配置とクリック時のショップ/職業変更/倉庫/強化への振り分け | GuiApi, ItemApi |
| [Dialogue](dialogue.md) | 分岐会話木とプレイヤーごとの永続フラグ | なし（DB/PlayerDataManagerのみ） |
| [Story](story.md) | 章/シナリオ進行とフラグによる複数エンディング解禁 | なし |
| [Dungeon](dungeon.md) | パーティ制限付きインスタンス型ダンジョン攻略 | StatusApi |
| [Region](region.md) | 名前付き矩形エリアと入退場メッセージ/自動ワープ | なし（永続化も無し） |
| [CutScene](cutscene.md) | カメラ移動/メッセージ/タイトル/エフェクトの時限シーケンス | EffectApi（ソフト） |
| [Event](event.md) | 季節/期間限定イベントとEXP/所持金の倍率 | なし |

## Config ファイル

`config.yml`, `quests.yml`, `npc.yml`, `dungeons.yml`, `dialogues.yml`, `story.yml`, `regions.yml`, `cutscenes.yml`, `events.yml`。詳細は[Configシステム](../architecture/config.md)。

## リロード

`/rpgworldadmin reload` で全モジュールの設定ファイルを再読込します。

## コマンド

| コマンド | 権限 | 用途 |
|---|---|---|
| `/rpgworldadmin reload` | `orelia.world.admin`（op） | 設定リロード |
| `/rpgquest list` / `/rpgquest abandon <id>` | `orelia.quest`（全員） | クエスト一覧/放棄 |
| `/dialoguechoice <index>` | `orelia.dialogue`（全員、内部利用） | チャットのクリック可能な選択肢から呼ばれる |
