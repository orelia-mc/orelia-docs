# NPC モジュール（`rpg.npc`）

configから静的な「疑似NPC」エンティティを配置し、右クリック時のインタラクションをショップ/職業変更/倉庫/強化（orelia-coreのGUI）またはクエスト受付画面（world側ローカル）へ振り分けます。

## ドメインモデル

```java
class NpcData {
    String id, name;
    NpcType type;
    EntityType entityType;
    String world; double x, y, z, yaw;
    List<String> dialogueLines;
    String conditionalItemId;
    List<String> conditionalDialogueLines;
    List<ShopEntry> shopStock;      // rpg.api.ShopEntry
    List<String> questIds;
    double enhancementCostBase, enhancementCostPerLevel;
}

enum NpcType {
    WEAPON_SHOP, ARMOR_SHOP, ACCESSORY_SHOP, QUEST_RECEPTIONIST,
    JOB_CHANGE, ENHANCEMENT, WAREHOUSE, GUILD_RECEPTIONIST, WEAPON_LEVELUP
}
```

`NpcData` の武器レベルアップ関連フィールド: `weaponLevelupItemMaterial`（消費するバニラ`Material`名）, `weaponLevelupItemAmount`, `weaponLevelupCostBase`, `weaponLevelupCostPerLevel`（`ENHANCEMENT`と同じ「基礎値 + 単価×現在レベル」の課金式）。

## `npc.yml` の実際のキー

```yaml
npcs:
  weapon_merchant:
    name: "武器商人ガレット"
    type: WEAPON_SHOP
    entity-type: VILLAGER
    world: world
    x: 10
    y: 64
    z: 10
    yaw: 180
    dialogue: ["いらっしゃい！腕利きの武器を揃えているよ。"]
    shop-stock:
      item_1: { kind: WEAPON, id: wooden_training_sword, price: 50 }
```

`shop-stock` の `kind`: `WEAPON` | `ACCESSORY` | `VANILLA`。`quest_receptionist` NPCは `conditional-item-id`, `conditional-dialogue`, `quest-ids: [...]` を持ちます。`blacksmith`（ENHANCEMENT）は `enhancement-cost-base`/`enhancement-cost-per-level` を持ちます。`weapon_master`（WEAPON_LEVELUP）は以下を持ちます：

```yaml
  weapon_master:
    name: "武器鍛錬師イグナス"
    type: WEAPON_LEVELUP
    entity-type: VILLAGER
    world: world
    x: -14
    y: 64
    z: 0
    yaw: 90
    dialogue: ["その武器、もっと磨いてやろうか？"]
    weapon-levelup-item-material: EMERALD
    weapon-levelup-item-amount: 5
    weapon-levelup-cost-base: 100
    weapon-levelup-cost-per-level: 50
```

未指定時のデフォルトは `NpcRepository`（`weapon-levelup-item-material: EMERALD`, `amount: 5`, `cost-base: 100`, `cost-per-level: 50`）で決まります。

## サービス

- **`NpcRepository`** — `load`/`findById`/`getAll`。追加で `add`/`replace`/`remove`（`/oladmin npc` の管理コマンドが使う、インメモリのみの変更）
- **`NpcKeys`** — 単一の `NamespacedKey("npc_id")` をラップ
- **`NpcSpawnService.spawn(id, Location)`/`despawn(id)`** — エンティティを無敵/サイレント/無重力/AI無しの固定物としてスポーン（`despawn`は除去）し、PDC `npc_id` タグを付与。`idOf`/`dataOf(Entity)`
- **`NpcSpawnSyncService.syncAll()`** — **サーバー起動時には自動実行されません**（旧仕様の「有効化1tick後に自動実行」は廃止され、コマンド駆動のみになりました — `NpcModule#onEnable` は現在 `syncAll()` を呼びません）。`/oladmin npc spawnall` から呼ばれたときのみ、`npc.yml` の全NPC（`JOB_CHANGE` タイプを除く）について、半径3ブロック以内にタグ付き済みエンティティが既にあるかをチェックしてから配置します（Bukkitはエンティティを永続化するため、再起動をまたいで生存— 重複スポーンを防ぐための対象チャンクの強制ロードも行う）。何度実行しても安全です。
- **`NpcAdminService`** — `/oladmin npc create|move|remove|list` の実装。`create`/`move` は配置系フィールド（name/type/entity-type/world/x/y/z/yaw）のみ `npc.yml` に書き戻し、`shop-stock`/`dialogue`/`quest-ids` 等は一切触りません（手書きコンテンツのまま）。エンティティのスポーン/デスポーンも同時に行います。

## NPCのスポーンはコマンド駆動のみ

過去バージョンではモジュール有効化時に `NpcSpawnSyncService` が自動実行され、NPCが重複スポーンするバグがありました（`241273c`で修正）。現在はNPCは**自動配置されません**。サーバー管理者は以下のいずれかを実行する必要があります：

- `/oladmin npc spawnall` — `npc.yml` の全NPC（`JOB_CHANGE` を除く）を一括配置（既存分はスキップ）
- `/oladmin spawnnpc <npc-id>` — 任意の1体を送信者の現在地に手動出現（`JOB_CHANGE` タイプNPCなど、`spawnall` の対象外のNPCを配置する唯一の方法）

## NPC管理コマンド（`/oladmin npc ...`）

`8c123a0` で追加。`NpcAdminCommand` → `/oladmin npc <create <id> <type> [entityType]|move <id>|remove <id>|list [page]|spawnall>`

- `create <id> <type> [entityType]` — 送信者（プレイヤーのみ）の現在地に新規NPCを作成。`entityType` 省略時は `VILLAGER`。既に同じ`id`が存在する場合は失敗。デフォルト値（`enhancement-cost-base: 100` 等）で作成され、`npc.yml` に書き戻された上で即座にスポーンされる
- `move <id>` — 既存NPC（定義＋実体）を送信者の現在地へ移動
- `remove <id>` — NPC定義・実体・`npc.yml` の該当セクションを完全削除
- `list [page]` — 全NPCを8件/ページで一覧表示（id/type/world/x/y/z）
- `spawnall` — 上記の通り、未配置NPCを一括配置

また `/oladmin spawnnpc <npc-id>` （`NpcSpawnCommand`）は送信者の現在地へ任意のnpc.yml定義済みIDを手動スポーンします。

## リスナー

`NpcInteractListener`（`PlayerInteractEntityEvent`） — セリフ送信（`conditional-item-id` によるオーバーライド有り）、`questProgressService.onNpcTalked` 呼び出し、その後 `NpcType` に応じて分岐：

- ショップ系 → `guiApi.openShop`
- `QUEST_RECEPTIONIST` → world側ローカルの `GuiManager` 経由で `QuestGuiScreen` を開く
- `JOB_CHANGE` → `guiApi.openJobChange`
- `WAREHOUSE` → `guiApi.openWarehouse`
- `ENHANCEMENT` → ローカルの `enhance()`（`ItemApi.identifyWeapon` で手持ち武器を確認、Vault経済で課金、`itemApi.enhanceWeapon` を呼ぶ）
- `WEAPON_LEVELUP` → ローカルの `levelUpWeapon()`（下記）
- `GUILD_RECEPTIONIST` → セリフ送信後、`NpcGuildInteractEvent` をfire（下記）

### 武器レベルアップNPC（`WEAPON_LEVELUP`）

`b5dff15` で新設。`levelUpWeapon(player, data)` の流れ：

1. `itemApi.identifyWeapon(手持ち武器)` が空なら `npc.weapon-levelup-need-weapon` を送って終了
2. `itemApi.getWeaponLevel(weapon)` で現在レベル、`itemApi.getWeaponLevelCap(playerId)` で上限を取得。既に上限なら `npc.weapon-levelup-cap-reached`
3. `data.getWeaponLevelupItemMaterial()` をバニラ `Material` としてパース（不正値なら何もせず終了 = fail closed）。プレイヤーが `weapon-levelup-item-amount` 分の素材を所持していなければ `npc.weapon-levelup-insufficient-material`
4. 費用 `weaponLevelupCostBase + weaponLevelupCostPerLevel * currentLevel` をVault経済でチェック（不足なら `npc.weapon-levelup-insufficient-money`）
5. 素材を消費・お金を引き落とし、`itemApi.levelUpWeapon(playerId, weapon)` を呼ぶ。負値が返れば（本来起こり得ないが）念のため上限到達メッセージにフォールバック
6. `itemApi.refreshWeaponLore(weapon)` でloreを更新し、`npc.weapon-levelup-success` を送信

`3b019db` で、素材不足メッセージに生の `Material` enum名（例: `DIAMOND_SWORD`）がそのまま表示されるバグを修正。`NpcInteractListener.prettifyMaterialName` がスネークケースの enum 名を `"Diamond Sword"` のようなタイトルケースに整形してから `npc.weapon-levelup-insufficient-material` の `material` プレースホルダーに渡すようになりました。

呼び出す orelia-core `ItemApi` メソッド: `identifyWeapon`, `getWeaponLevel`, `getWeaponLevelCap`, `levelUpWeapon`, `refreshWeaponLore`（強化用の `getEnhancementLevel`/`enhanceWeapon` とは別系統のAPI）。

### ギルドNPCフック（`GUILD_RECEPTIONIST`）

`b5dff15` で接続。orelia-worldはギルド機能を持たず（Guildは `orelia-extra` にあり、依存方向は `orelia-extra → orelia-world → orelia-core` のため、orelia-worldから`orelia-extra`へコンパイル依存できません）。そのため `GUILD_RECEPTIONIST` NPCと会話すると、`rpg.npc.event.NpcGuildInteractEvent`（`player`, `npcData` を保持する素朴なBukkitイベント）がfireされるだけです。`orelia-extra` がインストール・リスン（`rpg.extra.guild.listener.NpcGuildInteractListener`）していなければ無害なno-opです。

## 消費する orelia-core API

`GuiApi`, `ItemApi`（`identifyWeapon`/`getEnhancementLevel`/`enhanceWeapon`/`getWeaponLevel`/`getWeaponLevelCap`/`levelUpWeapon`/`refreshWeaponLore`）、加えて Vault `Economy`。`QuestModule` が既に登録済みであることに依存します（`plugin.getModuleManager().get(QuestModule.class)`）。

## WorldDebugApi

`8c123a0` で追加。orelia-world側の `rpg.world.api.WorldDebugApi` を `WorldApiModule` が Bukkit `ServicesManager` に公開します（orelia-coreの `rpg.api.DebugApi` と同じパターン）。`orelia-debug` プラグインが消費する想定で、config検査/編集の汎用メソッド（`listConfigFiles`, `getConfigValue`, `setConfigValue`, `saveConfig`, `describeConfigKeys`）に加え、world固有の2メソッドを持ちます：

- `forceCompleteQuestObjectives(UUID playerId, String questId)` — `QuestProgressService.forceCompleteObjectives` に委譲し、進行中クエストの全目標を強制達成
- `listNpcIds()` — 設定済み全NPC idをソート済みで返す（`NpcRepository` 由来）

実装 `WorldDebugApiImpl` は `WorldApiModule`（NPC/Questモジュールの後に有効化される）が `QuestModule`/`NpcModule` から依存を組み立てて登録します。
