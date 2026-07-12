# Accessory モジュール（`rpg.accessory`）

インベントリ下段に4つの専用スロット（護符/指輪/首飾り/翼）を割り当て、対応する種別のアイテムが物理的にそのスロットに入っている間だけステータス補正を適用します。

## ドメインモデル

```java
class AccessoryData {
    String id, name;
    AccessoryType type;
    StatSheet statBonus;
    List<String> description;
    int customModelData;
}

enum AccessoryType { CHARM, RING, NECKLACE, WING }
```

## `accessories.yml` の実際のキー

```yaml
accessories:
  guardian_charm:
    name: "守護の護符"
    type: CHARM
    stat-bonus: { HP: 30, DEF: 2 }
    description: ["&7最大HPと防御力を高める護符。"]
    custom-model-data: 200001
```

## サービス/マネージャー

- **`AccessoryRepository`** — `load`, `findById`, `getAll`
- **`AccessorySlotLayout`** — `PlayerInventory#getStorageContents()` 上の固定スロットマップ：`CHARM→27, RING→28, NECKLACE→29, WING→30`（31-35は予約）。`slotFor(type)`, `typeAtSlot(slot)`, `isAccessoryRow(slot)`（27-35）
- **`AccessoryFactory.create(AccessoryData)`** — 種別ごとの素材（CHARM→EMERALD, RING→GOLD_NUGGET, NECKLACE→AMETHYST_SHARD, WING→FEATHER）。PDCタグ `accessory_id`
- **`AccessoryIdentityService`** — `idOf`, `dataOf`
- **`AccessoryEffectService`**
    - `applyFromSlot(Player, type)` → `statusService.setEquipmentContribution(uuid, "accessory:" + type, statBonus)`
    - `clear(Player, type)`
    - `syncAll(Player)`（4種すべて）

## リスナー

`AccessorySlotListener`

- `onJoin` — 全スロットを同期
- `onClick` / `onDrag`（`InventoryClickEvent` / `InventoryDragEvent`） — アイテム種別がスロットの `AccessoryType` と一致しなければキャンセル（空気は常に許可）。変更は `schedulerService.runLater` で1tick後に反映

## 他モジュールへの依存

`status`（`StatSheet`/`StatType`, `StatusService` コントリビューション）のみ。加えて `util.ItemBuilder` とコアのスケジューラ。item/effect/skill への依存はありません。
