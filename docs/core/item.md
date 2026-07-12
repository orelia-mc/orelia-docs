# Item モジュール（`rpg.item`）

Config駆動の武器テンプレートシステム。`items.yml` から武器定義を読み込み、物理 `ItemStack` を生成し、所持アイテムをPDC（PersistentDataContainer）経由でテンプレートへ逆引きし、職業/レベル要件を検証します。

## ドメインモデル

```java
class WeaponData {
    String id, name;
    WeaponType weaponType;
    int weaponLevel;
    Rarity rarity;
    double attackPower;
    ElementType element;
    double critRate, critMultiplier;
    JobType requiredJob;      // nullable
    int requiredLevel;
    List<String> description;
    int customModelData;
    double sellPrice;
    int skillSlotCount;
}

enum Rarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY } // 各値が ChatColor を保持
enum WeaponType { SWORD, SPEAR, AXE, BOW }               // 各値が素材リストを保持
enum ElementType { NONE, FIRE, WATER, EARTH, WIND, LIGHT, DARK }
```

## `items.yml` の実際のキー

```yaml
weapons:
  wooden_training_sword:
    name: "見習いの剣"
    weapon-type: SWORD
    level: 1
    rarity: COMMON
    attack-power: 4.0
    element: NONE
    crit-rate: 5.0
    crit-multiplier: 1.5
    required-job: SWORDSMAN
    required-level: 1
    description: ["&7剣士見習いのための訓練用の剣。"]
    custom-model-data: 100001
    sell-price: 10.0
    skill-slot-count: 1
```

サンプル武器: `wooden_training_sword`, `flame_longsword`, `hunters_spear`, `berserkers_axe`, `hawkeye_bow`。

## サービス/マネージャー

- **`WeaponRepository`** — `load(YamlConfiguration)`, `findById(String)`, `getAll()`
- **`WeaponFactory.create(WeaponData)`** — `ItemBuilder` で `ItemStack` を構築し、PDC `weapon_id` タグを付与
- **`WeaponIdentityService`** — `idOf`, `dataOf`, `isOreliaWeapon`, `getEnhancementLevel`, `enhance`（PDC `enhancement_level` をインクリメント）, `enhancementMultiplier` = `1.0 + level*0.1`
- **`WeaponRequirementService.meetsRequirements(UUID, WeaponData)`** — 武器種の職業許可・必須職業・必須レベル（`StatusService` 経由）をチェック
- **`WeaponKeys`** — `NamespacedKey weaponId()`（`"weapon_id"`）, `enhancementLevel()`（`"enhancement_level"`）
- **`ItemManager`**（ファサード） — `createWeapon(String id)`, `findById`, `getAllWeapons`, `getIdentityService`, `getRequirementService`

## コマンド

`ItemCommand` → `/ol item give <player> <id> [amount]`（管理者向けスポナー。コード上の明示的な権限チェックなし）。

## 他モジュールへの依存

- **job** — `WeaponType`/`JobType` 要件チェック（`JobService` 経由）
- **status** — レベル要件チェック（`StatusService` 経由）
- **util** — `ItemBuilder`、クリティカル判定に `MathUtil.rollChance`（`WeaponUseListener` 内）

メタデータキー `orelia_skill_active`（`SKILL_OVERRIDE_METADATA`）は `skill` モジュールと連携し、スキル発動中は通常の武器ダメージ処理を抑制します。
