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

## リスナー

- **`SkillActivationListener`** — 右クリック=スロット0、シフト+右クリック=スロット1、オフハンド切替キー=スロット2（弓でも同様に動作）
- **`ArrowSkillDamageListener`** — 矢のダメージを倍率適用
- **`ExplosiveArrowHitListener`** — `ProjectileHitEvent` で起爆

## 他モジュールへの依存

- **item** — `WeaponType`, `WeaponData.attackPower`, `WeaponIdentityService.enhancementMultiplier`、`WeaponUseListener` 抑制用メタデータ
- **status** — `tryConsumeSp`
- **database** — 永続化

job / effect / economy / accessory への参照はありません。
