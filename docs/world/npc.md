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
    JOB_CHANGE, ENHANCEMENT, WAREHOUSE, GUILD_RECEPTIONIST
}
```

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

`shop-stock` の `kind`: `WEAPON` | `ACCESSORY` | `VANILLA`。`quest_receptionist` NPCは `conditional-item-id`, `conditional-dialogue`, `quest-ids: [...]` を持ちます。`blacksmith`（ENHANCEMENT）は `enhancement-cost-base`/`enhancement-cost-per-level` を持ちます。

## サービス

- **`NpcRepository`** — `load`/`findById`/`getAll`
- **`NpcKeys`** — 単一の `NamespacedKey("npc_id")` をラップ
- **`NpcSpawnService.spawn(id, Location)`** — エンティティを無敵/サイレント/無重力/AI無しの固定物としてスポーンし、PDC `npc_id` タグを付与。`idOf`/`dataOf(Entity)`
- **`NpcSpawnSyncService.syncAll()`** — 有効化1tick後に1回実行。半径3ブロック以内にタグ付き済みエンティティが既にあるかをチェックしてから配置（Bukkitはエンティティを永続化するため、再起動をまたいで生存）

## リスナー

`NpcInteractListener`（`PlayerInteractEntityEvent`） — セリフ送信（`conditional-item-id` によるオーバーライド有り）、`questProgressService.onNpcTalked` 呼び出し、その後 `NpcType` に応じて分岐：

- ショップ系 → `guiApi.openShop`
- `QUEST_RECEPTIONIST` → world側ローカルの `GuiManager` 経由で `QuestGuiScreen` を開く
- `JOB_CHANGE` → `guiApi.openJobChange`
- `WAREHOUSE` → `guiApi.openWarehouse`
- `ENHANCEMENT` → ローカルの `enhance()`（`ItemApi.identifyWeapon` で手持ち武器を確認、Vault経済で課金、`itemApi.enhanceWeapon` を呼ぶ）
- `GUILD_RECEPTIONIST` → セリフのみ（将来のギルド機能へのフック）

## 消費する orelia-core API

`GuiApi`, `ItemApi`、加えて Vault `Economy`。`QuestModule` が既に登録済みであることに依存します（`plugin.getModuleManager().get(QuestModule.class)`）。
