# Boss モジュール（`rpg.boss`）

既存の `monsters.yml` エントリの上に、HP閾値でのフェーズ演出・狂暴化ダメージ倍率・周期的なアビリティ発動を重ねます。モンスターのHP/攻撃力/ドロップのロジックを重複させません。

## ドメインモデル

```java
class BossData {
    String id, monsterId;
    List<BossPhase> phases;
    double enrageHpPercent, enrageDamageMultiplier;
    List<BossAbility> abilities;
}

class BossPhase {
    double hpThresholdPercent;
    String announceMessage;
}

class BossAbility {
    String id, name;
    BossAbilityType type;
    double damage, radius;
    int cooldownSeconds;
    String particle, sound, announceMessage;
}

enum BossAbilityType { AOE_SLAM, FIREBALL_BARRAGE }

class BossRuntimeState { // 永続化しない
    int phasesTriggered;
    boolean enraged;
}
```

## `bosses.yml` の実際のキー

```yaml
bosses:
  goblin_king_boss:
    monster-id: goblin_king
    enrage-hp-percent: 25
    enrage-damage-multiplier: 1.75
    phases:
      phase_1: { hp-threshold-percent: 70, announce-message: "&c&lゴブリンキングが激怒し始めた！" }
  flame_lord_boss:
    monster-id: flame_lord
    abilities:
      meteor_slam:
        name: "業火の一撃"
        type: AOE_SLAM
        damage: 18
        radius: 6
        cooldown-seconds: 12
        particle: FLAME
        sound: ENTITY_BLAZE_SHOOT
        announce-message: "&c業火の王が地面を叩きつけた！"
```

フェーズは読み込み時に閾値の降順でソートされます。デフォルト値: `enrage-hp-percent` 20.0、倍率 1.5、アビリティ種別 `AOE_SLAM`、ダメージ 10.0、半径 5.0、クールダウン15秒。

## サービス/マネージャー

- **`BossRepository`** — `load`, `findById`, `findByMonsterId`, `getAll`
- **`BossStateManager`** — `stateOf(UUID)`, `clear(UUID)`
- **`BossAbilityCastService(plugin, MonsterSpawnService, BossRepository)`**
    - `register(LivingEntity)`, `unregister(UUID)`
    - `tick()`（20tickごと） — 索敵範囲 `AGGRO_RANGE=24.0` 内で、ボスごとにクールダウン明けのアビリティを最大1つ発動
    - `castAoeSlam` — 近くのプレイヤーへ直接ダメージ
    - `castFireballBarrage` — プレイヤーごとに `SmallFireball` をスポーンし、メタデータ `orelia_boss_ability_fireball` = `double[]{damage, radius}` を付与
- **`BossModule.spawn(String bossId, Location)`** — `monsterModule.getSpawnService().spawn(...)` に委譲し、アビリティサービスへ登録

## リスナー

- **`BossEncounterListener`**（`EntityDamageEvent` MONITOR, `EntityDeathEvent`） — フェーズ演出（半径 `BROADCAST_RADIUS=48.0` 内へブロードキャスト + EXPLOSIONパーティクル）を発火し、`hp% <= enrageHpPercent` で狂暴化を設定
- **`BossEnrageListener`**（`LOW`、モンスターの攻撃ハンドラの後） — 攻撃者が狂暴化ボスならダメージに `enrageDamageMultiplier` を乗算
- **`BossFireballHitListener`** — `EntityExplodeEvent` からブロックダメージを除去し、`ProjectileHitEvent` で半径内のプレイヤーにダメージ

## 他モジュールへの依存

`monster`（`MonsterModule`, `MonsterSpawnService.spawn`/`idOf`）へのハード依存のみ。`skill`/`effect`/`item` へは**意図的に**依存しません（`BossAbility` のJavadocに、`SkillData` とは意図的に分離している旨の記載あり）。
