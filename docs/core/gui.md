# GUI モジュール（`rpg.gui`）

カスタムインベントリメニューの共通フレームワークと、実際のRPG画面（ステータス、装備、スキル、職業、ショップ、倉庫）。プロジェクトの規約上、GUI関連コードは全てこのパッケージに置きます。

## フレームワーク（`gui/framework/`）

```java
class Gui {
    Gui(String title, int size);
    Gui tag(String tag);              // 例: "warehouse"。クローズリスナーの分岐に使う
    Gui allowItemMovement();
    Gui set(int slot, GuiButton button);
    Optional<GuiButton> getButton(int slot);
    Inventory toInventory();          // Bukkit.createInventory(new GuiHolder(this), size, title) を遅延構築
}

class GuiButton {
    GuiButton(ItemStack icon, ClickAction action);
    static GuiButton display(ItemStack icon); // 装飾用・no-op

    interface ClickAction {
        void onClick(Player player, String clickType);
    }
}

class GuiHolder implements InventoryHolder {
    Gui getGui();
}

class GuiListener {
    // 全画面共通の単一の InventoryClickEvent ハンドラ
    // itemMovementAllowed でなければキャンセルし、クリックされたボタンのアクションへディスパッチ
}

class GuiManager {
    static void open(Player player, Gui gui); // player.openInventory(gui.toInventory())
}
```

## 画面（`gui/screen/`）

| 画面 | サイズ/タイトル | 内容 |
|---|---|---|
| `StatusGuiScreen` | 27, `"status"` | スロット4=プレイヤーヘッド（名前/レベル）。スロット10以降に各 `StatType` の値を、ローテーションするアイコン（REDSTONE, LAPIS_LAZULI, IRON_INGOT, SHIELD, FEATHER, ARROW, BOOK, EMERALD, BLAZE_POWDER, SUGAR）で表示 |
| `EquipmentGuiScreen` | 27, `"equipment"` | スロット13=手持ち武器（無ければBARRIER）。スロット19-22=各 `AccessoryType`（`AccessorySlotLayout` に従う） |
| `SkillGuiScreen` | 27, `"skill"` | 識別可能な武器が無ければBARRIERエラー。あれば武器種に対応する `SkillData` をスロット10以降にリスト表示。ENCHANTED_BOOK アイコンにレベル/最大/SPコスト/クールダウンを表示。左クリック=`progressService.upgradeSkill`、右クリック=`socketService.socket` |
| `JobGuiScreen` | 27, `"job"` | `JobType` ごとにスロット10から配置。現職業=GOLDEN_HELMET、他はLEATHER_HELMET。クリックで `jobService.changeJob` + クローズ |
| `ShopGuiScreen` | 54, `"shop"` | 呼び出し元が渡す `List<ShopEntry> stock` からスロット0-53を埋める。クリックで `economyService.withdraw` 後、`entry.kind()`（ACCESSORY/VANILLA/武器デフォルト）で解決してインベントリへ追加 |
| `WarehouseGuiScreen` | `WarehouseRepository.SIZE`（=54）, `"warehouse"`、`.allowItemMovement().tag("warehouse")` | `repository.load(uuid)` で保存済みの中身を直接ロード |

## `gui.yml` の実際のキー（タイトルのオーバーライドのみ、スロットごとのアイテム設定は無い）

```yaml
titles:
  status: "&8ステータス"
  equipment: "&8装備"
  skill: "&8武器スキル"
  job: "&8職業変更"
  quest: "&8クエスト"
  shop: "&8NPCショップ"
  warehouse: "&8倉庫"
```

`"quest"` キーは存在しますが、対応する画面クラスは orelia-core には無く、`orelia-world` 側にあります。

## コマンド

`StatusCommand` → `/ol status`（引数無し、プレイヤーのみ）で `StatusGuiScreen` を開く。

## リスナー

`WarehouseSaveListener` — `InventoryCloseEvent` で、holderが `GuiHolder` かつタグ `"warehouse"` なら `WarehouseRepository.save` で中身を保存。

## 他モジュールへの依存

`database, status, job, item, skill, accessory, economy` を必須とします。`StatusService`/`PlayerStatusComponent`、`JobService`/`JobType`、`ItemManager`/`WeaponIdentityService`/`WeaponType`、`SkillRepository`/`SkillProgressService`/`SkillSocketService`、`AccessorySlotLayout`/`AccessoryType`/`AccessoryRepository`/`AccessoryFactory`、`EconomyService`、および `rpg.api.ShopEntry` DTO と `util.ItemBuilder` を読みます。
