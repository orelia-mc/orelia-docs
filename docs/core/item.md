# Item モジュール（`rpg.item`）

Config駆動の武器テンプレートシステム。`items.yml` から武器定義を読み込み、物理 `ItemStack` を生成し、所持アイテムをPDC（PersistentDataContainer）経由でテンプレートへ逆引きし、職業/レベル要件を検証します。

## ドメインモデル

```java
class WeaponData {
    String id, name;
    WeaponType weaponType;
    int weaponLevel;          // items.yml の静的初期値（武器インスタンスの初期武器レベル）
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
    boolean unbreakable;      // デフォルト false。バニラの「壊れない」フラグ
}

enum Rarity { COMMON, UNCOMMON, RARE, EPIC, LEGENDARY } // 各値が ChatColor を保持
enum WeaponType { SWORD, SPEAR, AXE, BOW }               // 各値が素材リストを保持
enum ElementType { NONE, FIRE, WATER, EARTH, WIND, LIGHT, DARK }
```

`WeaponData.weaponLevel` は `items.yml` の `level:` から読み込まれる**武器種テンプレートの初期武器レベル**であり、個々の武器インスタンス（`ItemStack`）が持つ実際の武器レベルとは別物です。個体の武器レベルはPDCの `weapon_level` タグに保持され、初回読み取り時は未設定なら `WeaponData.weaponLevel` にフォールバックします（詳細は下記「武器レベルシステム」）。強化（enhancement、`enhancement_level` タグ）とは完全に独立した別概念です。

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
    required-job: FENCER
    required-level: 1
    description:
      - "&%7剣士見習いのための訓練用の剣。"
    custom-model-data: 100001
    sell-price: 10.0
    skill-slot-count: 1
    # unbreakable: true | false（省略時 false）— バニラの「壊れない」フラグ
```

`&%7` のようなカラーコードは `&0`-`&9`/`&a`-`&f` を独自の `&%` プレフィックス系コードへ一括置換したもの（#35）で、装飾コード（下線・取り消し線等）やhexコードはそのまま使えます。

サンプル武器: `wooden_training_sword`（FENCER, Lv1）, `flame_longsword`（FENCER, Lv10）, `hunters_spear`（職業不問, Lv5）, `berserkers_axe`（WARRIOR, Lv8）, `hawkeye_bow`（ARCHER, Lv6）。`hunters_spear` は `required-job` 未設定（職業不問）ですが、現在の職業ロースター（FENCER/WARRIOR/ARCHER）の `allowed-weapons` にSPEARが1つも無いため、実際にはどの職業でも武器ダメージが出ません（詳細は[検証手順](../verification/item.md)参照）。

## サービス/マネージャー

- **`WeaponRepository`** — `load(YamlConfiguration)`, `findById(String)`, `getAll()`
- **`WeaponFactory.create(WeaponData)`** — `ItemBuilder` で `ItemStack` を構築し、PDC `weapon_id` タグを付与。lore描画は `applyLore` に共通化されており、`refreshLore(ItemStack, WeaponData, WeaponIdentityService)` で武器レベル/強化/攻撃力の変更後に再描画できる
- **`WeaponIdentityService`** — `idOf`, `dataOf`, `isOreliaWeapon`, `getEnhancementLevel`, `enhance`（PDC `enhancement_level` をインクリメント）, `enhancementMultiplier` = `1.0 + level*0.1`, `getWeaponLevel`, `weaponLevelCap`, `levelUp`, `baseAttackPower`（下記「武器レベルシステム」参照）
- **`WeaponRequirementService.meetsRequirements(UUID, WeaponData)`** — 武器種の職業許可・必須職業・必須レベル（`StatusService` 経由）をチェック
- **`WeaponKeys`** — `NamespacedKey weaponId()`（`"weapon_id"`）, `enhancementLevel()`（`"enhancement_level"`）, `weaponLevel()`（`"weapon_level"`）
- **`WeaponLevelConfig`** — `config.yml: weapon-level.*` を読み込み、攻撃力ボーナス係数とレベル上限を提供
- **`ItemManager`**（ファサード） — `createWeapon(String id)`, `findById`, `getAllWeapons`, `getIdentityService`, `getRequirementService`, `refreshWeaponLore(ItemStack, WeaponData)`

## 武器レベルシステム（`weapon-level`）

`orelia-core` #27 で新設。既存の「強化」（enhancement、上限なし、`強化屋`NPC専用）とは別に、**武器レベル**という概念が追加されました。武器インスタンス（`ItemStack`）ごとにPDC `weapon_level` タグへ個別に記録され、`items.yml` の静的 `level:` を初期値として `WeaponIdentityService#levelUp` でプレイヤーが後から個別にレベルアップさせていきます。

### 基礎攻撃力の算出

`WeaponIdentityService.baseAttackPower(ItemStack, WeaponData)`:

```
weaponLevelBonus = 1.0 + weaponLevel * attackPowerFactor
baseAttackPower  = attackPower(items.yml) * weaponLevelBonus * enhancementMultiplier
```

`enhancementMultiplier`（既存の強化倍率）= `1.0 + enhancementLevel * 0.1` と乗算されるため、武器レベルと強化は独立に効果が積み重なります。この `baseAttackPower()` は `CombatDamageListener`（通常攻撃）と `SkillDamage`（武器スキル）の両方から呼ばれ、以前存在した個別のダメージ計算経路は統一されています。

### レベル上限（`weaponLevelCap`）

武器レベルはプレイヤー自身の**キャラクターレベル**（`StatusService`）に応じて上限が決まります（`WeaponLevelConfig#weaponLevelCap`）:

- プレイヤーレベルが `tier-step`（デフォルト10）未満: 上限は `initial-cap`（デフォルト5）固定。
- `tier-step` 以上: `floor(playerLevel / tier-step) * tier-step`（デフォルト設定ではLv10-19で上限10、Lv20-29で上限20、Lv30-39で上限30、…と無限に伸びる）。

`levelUp(ItemStack, WeaponData, int playerLevel)` は現在レベル+1が上限を超える場合 `-1` を返し、レベルは変化しません。

### `config.yml` の実際のキー

```yaml
weapon-level:
  # 武器レベル1ごとの基礎攻撃力ボーナス係数（例: 0.05 = レベルごとに+5%）
  attack-power-factor: 0.05
  # プレイヤーレベルが tier-step 未満のときの武器レベル上限
  initial-cap: 5
  # tier-step 以上では floor(playerLevel / tier-step) * tier-step が上限になる
  tier-step: 10
```

この設定追加に伴い `config-version` は `2` に更新されています（`ConfigManager` の自動マイグレーションが未知キーの追加・旧キーの警告を行う仕組みは[アーキテクチャ側のConfig章](../architecture/config.md)を参照）。

### lore表示

`WeaponFactory.applyLore` は現在の武器レベル（および強化中なら `(強化+N)`）と、計算後の攻撃力を1行で表示します。攻撃力は浮動小数点誤差（例: `37.699999999999996`）が出ることがあるため、**表示用にのみ**小数第1位へ丸められます（`String.format("%.1f", attackPower)`、基になる `double` 自体は丸められません、#35）。

## コマンド

`ItemCommand`（`AdminCommandRegistry` に登録、`/oladmin item ...`、`orelia.admin` 権限想定）:

- `/oladmin item give <player> <id> [amount]` — 管理者向けスポナー。コード上の明示的な権限チェックなし。
- `/oladmin item levelup` — 実行者（プレイヤーのみ）がメインハンドに持っている武器のレベルを、自分自身のキャラクターレベルに応じた上限まで1段階上げる。上限超過時は `item.weapon-level-capped` メッセージを返し変化なし。成功時は `WeaponFactory.refreshLore` でloreを即時再描画し `item.weapon-leveled-up` を返す。`orelia-world` 側の武器レベルアップNPC/GUIが実装されるまでの暫定的な手動/検証用エントリポイント。

## `rpg.api.ItemApi`（クロスモジュール連携）

`orelia-world` の武器レベルアップNPCなど、`orelia-core` 外から武器を操作する唯一の公式手段です:

- `createWeapon`, `identifyWeapon`, `getAllWeaponIds`, `weaponMeetsRequirements`, `getEnhancementLevel`, `enhanceWeapon` — 既存機能
- `getWeaponLevel(ItemStack)` — 現在の武器レベルを返す（未識別の武器は `0`）
- `getWeaponLevelCap(UUID playerId)` — そのプレイヤーが現在レベルアップさせられる上限を返す。NPC側はチャージ（経済コスト徴収）前にこれで上限チェックが可能
- `levelUpWeapon(UUID playerId, ItemStack stack)` — `playerId` のキャラクターレベルでゲートしてレベルアップを試行。新レベル、または上限超過時は `-1` を返す
- `refreshWeaponLore(ItemStack stack)` — `levelUpWeapon`/`enhanceWeapon` 呼び出し後にloreを即時再描画する。このモジュールが武器と認識しないスタックに対してはno-op

## 他モジュールへの依存

- **job** — `WeaponType`/`JobType` 要件チェック（`JobService` 経由）
- **status** — レベル要件チェック、武器レベルアップ時のプレイヤーレベル参照（`StatusService` 経由）
- **util** — `ItemBuilder`、クリティカル判定に `MathUtil.rollChance`（`WeaponUseListener` 内）

メタデータキー `orelia_skill_active`（`SKILL_OVERRIDE_METADATA`）は `skill` モジュールと連携し、スキル発動中は通常の武器ダメージ処理を抑制します。

ダメージ計算そのもの（武器ヒット → ATK% → DEF → クリティカル → 弱点属性）は `rpg.monster.listener.CombatDamageListener` に単一化されており（#26）、`monster` モジュールが `item`/`status` の両方より後に有効化される都合上、そちらに配置されています。武器の基礎攻撃力（本ページの `baseAttackPower()`）はそのパイプラインの入力の一つです。
