# Skill モジュール（`rpg.skill`）

Config駆動の武器スキルシステム。`skills.yml` の定義をプラグ可能な「executor」アーキタイプが実行し、武器の `ItemStack` にスキルをソケットし、スキルポイントによる習得/レベルアップを管理します。

## ドメインモデル

```java
class SkillData {
    String id, name;
    WeaponType weaponType;
    String executorType;
    double spCost, cooldownSeconds, damageMultiplier;
    String effectParticle;
    double range, radius, knockback;
    int maxLevel;

    // scaledDamageMultiplier(level) = damageMultiplier * (1 + 0.1 * max(0, level - 1))
}

class PlayerSkillComponent implements PlayerDataComponent {
    UUID owner;
    int skillPoints;
    Map<String, Integer> skillLevels;
    Map<String, Long> cooldownExpiry; // 実行時のみ、永続化しない
}
```

## `skills.yml` の実際のキー

```yaml
slash:
  name: "斬撃"
  weapon-type: SWORD
  executor-type: MELEE_CONE
  sp-cost: 8
  cooldown-seconds: 3
  damage-multiplier: 1.4
  effect-particle: SWEEP_ATTACK
  range: 4.0
  radius: 0
  knockback: 0.3
  max-level: 5
```

全15スキル定義。`executor-type` の値: `MELEE_CONE`, `MELEE_AOE`, `DASH_STRIKE`, `ARROW_VOLLEY`, `EXPLOSIVE_ARROW`。

## サービス/マネージャー/Executor

- **`SkillRepository`** — `load`, `findById`, `getAll`, `getByWeaponType(WeaponType)`
- **`PlayerSkillRepository`**（`SchemaOwner`） — テーブル `player_skill_points`, `player_skill_level`
- **`SkillExecutorRegistry`** — `register(String, SkillExecutor)`, `get(String) -> Optional<SkillExecutor>`
- **`SkillCastService.cast(Player caster, String skillId)`** → `Optional<CastFailure>`

    ```java
    enum CastFailure {
        UNKNOWN_SKILL, WRONG_WEAPON, NOT_SOCKETED,
        NOT_LEARNED, ON_COOLDOWN, NOT_ENOUGH_SP, NO_EXECUTOR
    }
    ```

    武器種一致 → ソケット有無 → 習得レベル>0 → クールダウン → SPコスト（`StatusService.tryConsumeSp`）の順にチェックし、executor へディスパッチ。

- **`SkillProgressService`** — `getSkillLevel`, `grantSkillPoints`, `upgradeSkill(UUID, String)`（SP1消費、+1レベル、`maxLevel` で頭打ち）
- **`SkillSocketService`** — スキルはPDC文字列 `"socketed_skills"` に `;` 区切りで保存。`getSocketedSkills`, `hasSkill`, `socket(ItemStack, String, int maxSlots)`, `unsocket`

### Executor インターフェース

```java
interface SkillExecutor {
    void execute(Player caster, SkillData data, int skillLevel);
}
```

実装: `MeleeConeExecutor`, `MeleeAoeExecutor`, `DashStrikeExecutor`（前方へ速度付与）, `ArrowVolleyExecutor`（1本または3本拡散、メタデータ `orelia_skill_arrow_multiplier`）, `ExplosiveArrowExecutor`（メタデータ `orelia_skill_explosive_arrow` → `double[]{amount, radius}`）

### `SkillDamage` — 近接3系executorの共通ダメージヘルパー

`MeleeConeExecutor`/`MeleeAoeExecutor`/`DashStrikeExecutor` は `rpg.skill.executor.SkillDamage` を経由してダメージを与えます（`ArrowVolleyExecutor`/`ExplosiveArrowExecutor` は矢を飛ばすだけで、実ダメージは別リスナーが担当 — 下記参照）。

- **`baseDamage(caster, data, skillLevel)`** — `WeaponIdentityService.baseAttackPower(weapon, weaponData)`（武器がない/識別できない場合は `1.0`）× `data.scaledDamageMultiplier(skillLevel)` を求め、`DamageFormula.applyAttackBonus` でキャスターのATK%まで乗せた値を返す。**DEF軽減・クリティカル判定・属性弱点はまだ適用されていない** — これらは対象ごとに異なるため、実際にダメージが飛ぶ瞬間に `rpg.monster.listener.CombatDamageListener` が個別に解決する（AOE/コーン系スキルが複数対象へ同じ基礎ダメージを一度だけ計算し、対象ごとの防御力差だけをそれぞれ反映するため）。`WeaponIdentityService.baseAttackPower` 自体は武器の `attack-power`（`items.yml`）に武器レベルボーナス（`config.yml: weapon-level.attack-power-factor` 分、詳細は [Item モジュール仕様](item.md)）と強化倍率（`enhancementMultiplier`）を掛けたもの。
- **`apply(caster, target, amount)`** — キャスターに `DamageFormula.SKILL_OVERRIDE_METADATA`（`"orelia_skill_active"`）メタデータを立ててから `target.damage(amount, caster)` を呼ぶ。このメタデータにより `CombatDamageListener` は「base attack power + ATK%までは計算済み」と判断し、DEF/クリティカル/弱点だけを解決する（通常武器攻撃力での上書きを防ぐ）。

### 矢系スキルのダメージ適用

- **`ArrowVolleyExecutor`** は矢に `orelia_skill_arrow_multiplier` メタデータだけを付け、実ダメージ加算は `ArrowSkillDamageListener`（`LOW` 優先度）が矢がエンティティに当たった `EntityDamageByEntityEvent` を捕まえて `event.setDamage(event.getDamage() * multiplier)` で行う。
- **`ExplosiveArrowExecutor`** は矢に `orelia_skill_explosive_arrow` メタデータ（`double[]{amount, radius}`）を付け、`ExplosiveArrowHitListener`（`ProjectileHitEvent`）が着弾時にAoEダメージを与える。このとき射手（キャスター）に `SkillDamage.apply` と同じ `SKILL_OVERRIDE_METADATA` を一時的に立ててから `target.damage(amount, caster)` を呼ぶ — これを付けないと `CombatDamageListener` がプレイヤー発ダメージとみなしてキャスターの手持ち武器の通常攻撃力で上書きしてしまうバグが過去にあった（爆裂矢の実ダメージが武器攻撃力で上書きされる不具合、修正済み）。

## リスナー

- **`SkillActivationListener`** — 右クリック=スロット0、シフト+右クリック=スロット1、オフハンド切替キー=スロット2（弓でも同様に動作）
- **`ArrowSkillDamageListener`** — `multi_shot`/`power_shot` の矢のダメージへ倍率を適用
- **`ExplosiveArrowHitListener`** — `ProjectileHitEvent` で起爆、`SKILL_OVERRIDE_METADATA` を付与してから `CombatDamageListener` にDEF/クリティカル/弱点だけを解決させる

## 他モジュールへの依存

- **item** — `WeaponType`, `WeaponData.attackPower`, `WeaponIdentityService.baseAttackPower`（武器レベル・強化を織り込んだ攻撃力）, `WeaponIdentityService.enhancementMultiplier`
- **status** — `tryConsumeSp`, `StatusService.getFinalStats`（ATK%の取得）
- **monster** — `rpg.status.combat.DamageFormula`／`CombatDamageListener` によるダメージ確定（DEF/クリティカル/属性弱点の適用は skill モジュール外）
- **database** — 永続化

job / effect / economy / accessory への参照はありません。
