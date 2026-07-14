# GUI（各種画面）の動作確認

対象: `rpg.gui`（[GUI モジュール仕様](../core/gui.md)）。カスタムインベントリメニューの共通フレームワークと、実際のRPG画面（ステータス、装備、スキル、職業、ショップ、倉庫）です。

## 到達経路（現状の実際の導線）

各画面への到達経路は画面ごとに異なります。検証前に、どの画面がどこから開けるかを正しく把握してください。

| 画面 | サイズ/タイトル | 到達経路（現状） |
|---|---|---|
| `StatusGuiScreen` | 27, "status" | `/ol status`（`StatusCommand`）から直接開ける |
| `EquipmentGuiScreen` | 27, "equipment" | `orelia-core`/`orelia-world` のどちらのコードからも `GuiApi.openEquipment` を呼んでいない（**未接続のまま**） |
| `SkillGuiScreen` | 27, "skill" | `orelia-core` 単体では未接続。`orelia-world` 導入時は「プレイヤー情報」ネザースター（右クリック）→「スキル」タブ経由で `GuiApi.openSkill` が呼ばれ到達できる（詳細は[Skill の確認手順](skill.md)） |
| `JobGuiScreen` | 27, "job" | `orelia-world` の `JOB_CHANGE` 型NPC（`job_master`、要 `/oladmin spawnnpc job_master` での手動配置）から `guiApi.openJobChange` 経由（[Job の確認手順](job.md)） |
| `PlayerInfoGuiScreen`（`orelia-world` 側） | 27, "プレイヤー情報" | `orelia-world` 導入時、ホットバー右端に固定されるネザースターの右クリックで開く。クエスト/ジョブ/スキル/実績(仮)の4カテゴリボタンを持つ入口メニュー |
| `ShopGuiScreen` | 54, "shop" | `orelia-world` のショップNPC（`WEAPON_SHOP`/`ARMOR_SHOP`/`ACCESSORY_SHOP`）から `guiApi.openShop` 経由 |
| `WarehouseGuiScreen` | `WarehouseRepository.SIZE`(=54), "warehouse" | `orelia-world` の `WAREHOUSE` 型NPCから `guiApi.openWarehouse` 経由 |

## 1. フレームワーク共通挙動の確認

どの画面でも共通のはずの `GuiListener`/`Gui`/`GuiButton` の挙動です。

1. `allowItemMovement()` を呼んでいない画面（`status`/`equipment`/`skill`/`job`/`shop`）では、GUI内でアイテムを自由に動かせない（`InventoryClickEvent` がキャンセルされる）ことを確認する。
2. `GuiButton.display(...)`（装飾用・no-op）のボタンをクリックしても何も起きないことを確認する（`StatusGuiScreen` の統計表示アイコンなど）。
3. `WarehouseGuiScreen` は `.allowItemMovement().tag("warehouse")` 付きなので、GUI内でアイテムを自由に動かせることを確認する。

## 2. `StatusGuiScreen` の確認

```
/ol status
```

1. 27スロットのGUIが開き、タイトルが `gui.yml: titles.status`（デフォルト「ステータス」）どおりであること。
2. スロット4にプレイヤーヘッド（名前/レベル表示）があること。
3. スロット10以降に各 `StatType`（HP, SP, ATK, DEF, CRT, CRT_DMG, SPD の7種。AGI/DEX/INTは廃止済み）の値が、7種のアイコン（REDSTONE, LAPIS_LAZULI, IRON_INGOT, SHIELD, EMERALD, BLAZE_POWDER, SUGAR）付きで表示されること。
4. [Job](job.md)/[Accessory](accessory.md)/[Status](status.md)の各確認手順で変化させたステータスが、ここに正しく反映されていること。

## 3. `EquipmentGuiScreen` の確認

到達経路が現状無いため、検証用の導線（`GuiApi.openEquipment` を叩く一時コマンド等）を用意して確認します。

1. スロット13に手持ち武器が表示される（無ければBARRIER）こと。
2. スロット19-22に `AccessorySlotLayout` に従った各 `AccessoryType` のアイテムが表示されること（実際のインベントリ下段スロット27-30と連動しているか確認）。

## 4. `SkillGuiScreen` の確認

[Skill の確認手順](skill.md)を参照してください。`orelia-world` 導入時の到達経路（プレイヤー情報ネザースター経由）と、`orelia-core` 単体での回避策をそちらにまとめています。

## 5. `JobGuiScreen` の確認

[Job の確認手順](job.md)を参照してください。

## 5.5. `PlayerInfoGuiScreen`（プレイヤー情報ネザースター、`orelia-world` 導入時）の確認

1. サーバー参加時、ホットバー右端のスロットに「プレイヤー情報」ネザースターが自動で入っていること。
2. このアイテムを投げる（Qキー）・ドラッグで動かす・クリエイティブモードで削除する、いずれの操作もキャンセルされ、常に同じスロットに固定され続けることを確認する（`PlayerInfoItemListener` の各ハンドラ）。リスポーン後も再付与されること。
3. 右クリックで27スロットの「プレイヤー情報」メニューが開き、スロット10/12/14/16に「クエスト」「ジョブ」「スキル」「実績」の4ボタンが均等配置されていること。
4. 「ジョブ」ボタン → 現在の職業名（未就業なら「未就業」+BARRIER表示）だけを表示する読み取り専用画面が開くこと（**ここから職業変更はできない**。職業変更は引き続き `JOB_CHANGE` NPC経由のみ、[Job の確認手順](job.md)参照）。
5. 「スキル」ボタン → 習得済みスキル一覧（無ければ「習得済みスキルはありません」の案内ボタン）が表示され、いずれかをクリックすると `orelia-core` の `SkillGuiScreen` が開くこと（[Skill の確認手順](skill.md)）。
6. 各サブ画面の右下（36スロット中スロット35）に「戻る」ボタンがあり、クリックでこのメニューに戻ること。
7. 「実績」ボタン → プレースホルダー画面（`PlayerInfoAchievementGuiScreen`）が開くこと。中身の実装はまだ無いため、開けること自体を確認できれば十分です。

## 6. `ShopGuiScreen` の確認（`orelia-world` 導入時）

1. ショップNPCに話しかけ、54スロットのGUIが `npc.yml` の `shop-stock` どおりのラインナップで開くことを確認する。
2. クリックすると `economyService.withdraw` が呼ばれ、残高不足なら購入が失敗すること（[Economy の確認手順](economy.md)）。
3. `entry.kind()` が `WEAPON`（デフォルト）/`ACCESSORY`/`VANILLA` それぞれで正しい種類のアイテムがインベントリに追加されること。

## 7. `WarehouseGuiScreen` の確認（`orelia-world` 導入時）

1. `WAREHOUSE` 型NPCに話しかけ、54スロットの倉庫GUIが開くこと。
2. アイテムを入れて閉じると `WarehouseSaveListener`（`InventoryCloseEvent`、holderが `GuiHolder` かつタグ `"warehouse"`）により `WarehouseRepository.save` で保存されること。
3. 再度開いたときに、保存した内容がそのまま復元されていること。
4. 別プレイヤーの倉庫と混同していないこと（プレイヤーごとに独立していること）。

## 8. `gui.yml` タイトル上書きの確認

`gui.yml` の `titles.*` を書き換えて `/oladmin reload` した後、各画面のタイトルが変更後の値に反映されることを確認します（スロットごとのアイテム配置はconfig化されていないため、レイアウト自体は変わりません）。`titles.quest` キーは存在しますが対応する画面クラスが `orelia-core` に無い（`orelia-world` 側にある）ため、`orelia-core` 単体ではこのキーの変更が反映される場所がないことも確認しておいてください。

## 異常系・エッジケース

- GUIを開いた状態でサーバーからログアウトし、再ログインしてもクラッシュ・アイテム消失が起きないこと。
- `status`/`skill`/`job` など `allowItemMovement()` 無しの画面で、Shift+クリックや数字キーでのホットバー移動を試み、いずれもキャンセルされること。
