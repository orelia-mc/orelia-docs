# Accessory（装飾品）の動作確認

対象: `rpg.accessory`（[Accessory モジュール仕様](../core/accessory.md)）。インベントリ下段の4つの専用スロット（護符/指輪/首飾り/翼）に対応種別のアイテムが**物理的に入っている間だけ**ステータス補正が乗ります。

## 前提条件

`accessories.yml` 標準定義：

| ID | 表示名 | 種別 | ステータス補正 |
|---|---|---|---|
| `guardian_charm` | 守護の護符 | CHARM | HP +30, DEF +2 |
| `warriors_ring` | 戦士の指輪 | RING | ATK +3, CRT +1 |
| `mages_necklace` | 賢者のネックレス | NECKLACE | SP +20 |
| `swift_wing` | 疾風の羽 | WING | （ステータス補正なし、移動速度用） |

スロット対応（`PlayerInventory#getStorageContents()` 上）: `CHARM→27, RING→28, NECKLACE→29, WING→30`（31-35は予約、未使用）。

## 1. スロット制約の確認

1. アクセサリ用アイテムを取得する（`/ol item give` はItemモジュール専用なので、`AccessoryFactory.create` 経由のアイテムはショップNPC（`ACCESSORY_SHOP`）や、テスト用に別途生成手段が必要です）。
2. `guardian_charm`（CHARM）をスロット27（インベントリ下段の該当位置）以外の一般スロットには通常どおり置けることを確認。
3. `guardian_charm` をスロット27に置くと成功し、`warriors_ring`（RING）をスロット27に置こうとするとキャンセルされる（種別不一致）ことを確認する。
4. 逆にスロット27を**空**にする操作（アイテムを取り除く）は常に許可されることを確認する。
5. `InventoryDragEvent`（ドラッグでのアイテム移動）でも同様に種別チェックが効くことを確認する。

## 2. ステータス補正の反映確認

`AccessoryEffectService.applyFromSlot` は `statusService.setEquipmentContribution(uuid, "accessory:" + type, statBonus)` を呼びます。

1. 装着前に `/ol status` でHP/DEF/ATK/CRT/SPの最大値をメモする。
2. `guardian_charm` をスロット27（CHARM）に置く。
3. 1tick後（`schedulerService.runLater` で反映）に `/ol status` を開き直し、HP最大値+30、DEF+2になっていることを確認する。
4. 同様に `warriors_ring`（RING, スロット28）でATK+3・CRT+1、`mages_necklace`（NECKLACE, スロット29）でSP最大値+20を確認する。
5. 4種すべてを同時装着し、補正が加算で乗ること（他ソースの補正、例えばJobのパッシブと共存すること）を確認する。
6. アクセサリをスロットから取り除くと、`clear` により該当補正だけが消え、他の補正（他アクセサリ・職業パッシブ）は残ることを確認する。

## 3. join時の再同期確認

`AccessorySlotListener.onJoin` は全スロットを同期します。

1. 4種のアクセサリを装着した状態でサーバーから退出する。
2. 再参加し、`/ol status` の補正値が退出前と同じであることを確認する（アイテム自体はインベントリに物理的に残っているため、通常のBukkitインベントリ永続化に依存）。

## 異常系・エッジケース

- 空気（何も持っていない状態）をアクセサリスロットに「置く」操作は常に許可されること（`isAccessoryRow` の27-35全体、31-35は予約スロットとして常に許可されるか個別に確認）。
- 種別が一致しないアイテムをスロットへ強制的に入れようとするクライアント側の不整合操作（例: ドラッグ&ドロップの端数移動）でもサーバー側でキャンセルされ、クライアント表示とサーバー状態がズレないこと。
- `swift_wing`（WING）は `stat-bonus` が未設定のため、装着してもステータスGUI上の数値変化が無いことを確認する（見た目や移動速度など、ステータス以外の効果がある場合はそちらで確認）。
