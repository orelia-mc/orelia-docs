# Monster モジュール（`rpg.monster` + `rpg.monster.spawnpoint`）

Config駆動のモンスターテンプレート（`monsters.yml`）。バニラエンティティをスポーンさせ、被弾時に戦闘ステータスを適用し、撃破時に経験値/お金/ドロップを報酬として与えます。

`spawnpoint` サブモジュールは、管理者が設置した永続スポーン地点を管理し、一定間隔で最大同時湧き数の上限までモンスターをスポーンさせます（バニラの自然湧き/スポナー湧きを置き換え）。

## ドメインモデル

```java
class MonsterData {
    String id, name;
    EntityType entityType;
    double hp, attackPower, defense;
    ElementType element;
    ElementType weakness;      // NONE=弱点なし。一致する属性の武器で攻撃すると被ダメージ x1.5
    AiType aiType;
    List<DropEntry> drops;
    long expReward;
    double moneyMin, moneyMax;
    List<MonsterAbility> abilities; // 通常モンスターの能動アビリティ（後述）
    double critRate, critMultiplier; // 省略時 0（常時非クリティカル）/1.5
}

class MonsterAbility {
    String id, name;
    MonsterAbilityType type;   // AOE_SLAM | FIREBALL_BARRAGE（bossパッケージのBossAbilityと同形）
    double damage, radius;
    int cooldownSeconds;
    String particle, sound, announceMessage;
}

class DropEntry {
    String weaponId, vanillaMaterial;
    double chancePercent;
    int minAmount, maxAmount;
}

enum AiType { PASSIVE, AGGRESSIVE, RANGED }

class MonsterSpawnPoint {
    UUID id;
    String monsterId, world;
    double x, y, z;
    int intervalSeconds, maxAlive;
}
```

## `monsters.yml` の実際のキー

```yaml
monsters:
  forest_slime:
    name: "森のスライム"
    entity-type: SLIME
    hp: 15
    attack-power: 2
    defense: 0
    element: NONE
    weakness: FIRE       # element: FIRE の武器で攻撃すると被ダメージ x1.5（DEF軽減後に乗算）
    ai-type: AGGRESSIVE
    exp-reward: 5
    money-min: 1
    money-max: 3
    drops:
      slime_ball: { vanilla-material: SLIME_BALL, chance-percent: 60, min-amount: 1, max-amount: 2 }

  goblin_raider:
    name: "ゴブリンの略奪者"
    entity-type: ZOMBIE
    hp: 40
    attack-power: 5
    defense: 2
    element: NONE
    ai-type: AGGRESSIVE
    exp-reward: 15
    money-min: 5
    money-max: 12
    drops:
      rotten_flesh: { vanilla-material: ROTTEN_FLESH, chance-percent: 80, min-amount: 1, max-amount: 2 }
      training_sword: { weapon-id: wooden_training_sword, chance-percent: 5, min-amount: 1, max-amount: 1 }
    abilities:                 # 通常モンスターの能動アビリティ（MonsterAbilityCastService、後述）
      war_cry_slam:
        name: "雄叫びの一撃"
        type: AOE_SLAM
        damage: 6
        radius: 4
        cooldown-seconds: 10
        particle: SWEEP_ATTACK
        sound: ENTITY_ZOMBIE_ATTACK_IRON_DOOR
        announce-message: "&cゴブリンの略奪者が雄叫びを上げた！"

  goblin_king:
    name: "ゴブリンキング"
    entity-type: ZOMBIE
    hp: 400
    attack-power: 12
    defense: 5
    element: NONE
    ai-type: AGGRESSIVE
    exp-reward: 200
    money-min: 100
    money-max: 250
    drops:
      flame_longsword: { weapon-id: flame_longsword, chance-percent: 15, min-amount: 1, max-amount: 1 }
      emerald: { vanilla-material: EMERALD, chance-percent: 100, min-amount: 3, max-amount: 8 }
    # abilities は設定しない — bosses.yml の monster-id からも参照されるモンスターIDには
    # abilities を追加しない運用ルール（詳細は下記）
```

`weakness` は省略時 `NONE`（弱点なし）。`crit-rate`（省略時0=常時非クリティカル）・`crit-multiplier`（省略時1.5、`crit-rate > 0` のときのみ意味を持つ）も任意キーとして追加できます。`abilities` は `bosses.yml` の `abilities` ブロックと同じ形（`type: AOE_SLAM | FIREBALL_BARRAGE`）です。`monsters.yml` のコメントは、`bosses.yml` の `monster-id` として参照されているモンスターID（`goblin_king`, `flame_lord`）に `abilities` を設定しないよう明記しています——`MonsterAbilityCastService` への登録は `BossModule.spawn()` を経由しない湧き経路（スポーンポイント、`/oladmin monster spawn`）でのみ行われるため実害はありませんが、意図が分かりにくくなるための運用上の注意です。

## サービス

- **`MonsterRepository`** — `load`, `findById`, `getAll`
- **`MonsterSpawnService`**
    - `spawn(String monsterId, Location)` / `spawn(..., spawnPointId)` — バニラエンティティをスポーンし、PDCタグ `monster_id`/`spawn_point_id` を付与。攻撃力/防御力はバニラ属性ではなく**リスナー内で被弾ごとに適用**（後述の `CombatDamageListener`）
    - `idOf`, `spawnPointIdOf`, `dataOf`
    - `scaledCurrentHpOf`, `applyScaledCombatDamage`, `applyEnvironmentalDamage`, `updateHealthBar` — Scaled Health（下記）まわりの帳簿管理
- **`MonsterDropService(ItemManager, EconomyService, StatusService).rewardKiller(data, killer, location)`** — `statusService.addExperience`、`economyService.deposit(MathUtil.lerp(moneyMin, moneyMax, random))`、各 `DropEntry` を `MathUtil.rollChance` で判定し `itemManager.createWeapon` またはバニラ素材で自然ドロップ
- **`MonsterKeys`** — PDCキー `monster_id`, `spawn_point_id`, `scaled_current_hp`
- **`MonsterAbilityCastService(plugin, MonsterSpawnService)`** — 通常モンスターの能動アビリティ（後述）
- **`MonsterSpawnPointService`** — `loadAll()`, `add(Player admin, monsterId, intervalSeconds, maxAlive)`, `remove(UUID)`, `getAll()`, `tick()`（20tickごとにチェック） — `aliveCount >= maxAlive` または未到来ならスキップ、それ以外はスポーンして記録。`onEntityRemoved` でスロットを解放
- **`MonsterSpawnPointManager`** — 実行時の帳簿管理。`isDueToSpawn` = `now - lastSpawn >= intervalSeconds * 1000`
- **`MonsterSpawnPointRepository`**（`SchemaOwner`） — テーブル `monster_spawn_point(id, monster_id, world, x, y, z, interval_seconds, max_alive)`

## Scaled Health（モンスターの体力スケーリング）

`monsters.yml` の `hp`（フレイムロードのようなボス格だと数百〜数千まで届く）は、バニラの `Attribute.MAX_HEALTH` に安全に収まる範囲を超え得ます。そのため `MonsterSpawnService.spawn` は `Attribute.MAX_HEALTH` を `min(data.getHp(), config.yml: combat.scaled-health.vanilla-cap)`（デフォルト1024）に設定するだけで、**モンスターの本当の現在HPは別途PDC（`MonsterKeys.scaledCurrentHp()`）で管理**します。プレイヤー側の同じ仕組みは [Status モジュール仕様の Scaled Health](status.md#scaled-health-プレイヤーの体力スケーリング) を参照してください。

- `CombatDamageListener` が被弾ごとに `MonsterSpawnService.applyScaledCombatDamage(entity, data, scaledDamage)` を呼び、スケール済み現在HPだけを減算する（バニラ体力は同リスナーが `ScaledHealthService.convertDamageToVanilla` で変換した量を `event.setDamage` に渡すことで、Bukkit標準のダメージ処理（ノックバック・被弾音・死亡判定）を通して更新される）。
- 落下ダメージ等 `EntityDamageByEntityEvent` を経由しない環境ダメージは `MonsterHealthBarListener` から `MonsterSpawnService.applyEnvironmentalDamage` が呼ばれ、実際に減ったバニラ体力の割合をスケール済みHP側にもミラーする。
- `scaledCurrentHpOf` はPDC値が無い（このシステム導入前にスポーンした個体）場合 `data.getHp()`（満タン）にフォールバックする。

## HPバー・ダメージ数値表示

- **`MonsterHealthBarListener`**（`EntityDamageEvent`、`MONITOR` 優先度） — 被弾のたびにネームタグを `MonsterSpawnService.updateHealthBar` でHPバーとして再描画する。`config.yml: monster.health-bar`（`enabled`（デフォルトtrue）, `length`（10）, `format`（`"{name} &%7[{bar}&%7] &%f{current}/{max}"`）, `filled-color`（`&%a`）, `empty-color`（`&%8`）) で見た目を制御できる。`enabled: false` ならバー無しの通常名前のみ表示。
- **`DamageDisplayListener`**（`EntityDamageEvent`、`MONITOR` 優先度） — 命中位置にフローティングのダメージ数値を表示する（モンスター・プレイヤー双方対象）。数値は表示用エンティティで、`config.yml: combat.damage-display`（`enabled`, `duration-ticks`（20）, `gravity-per-tick`（0.02、重力で落下しながら消える演出）, `y-offset`, `normal-color`, `crit-color`, `crit-scale`（1.3）) で制御。Scaled Health対象の被害者では、バニラ換算後の小さい値ではなく元のスケール済みダメージ量（`DamageFormula.SCALED_DAMAGE_METADATA_KEY`）を表示する。クリティカルかどうかは `CombatDamageListener` が付与した `DamageFormula.CRIT_METADATA_KEY` から読む。

## 弱点属性（Elemental Weakness）

`monsters.yml` の `weakness`（省略時 `NONE`）は、攻撃者の手持ち武器の `element`（`items.yml`）が一致した場合にのみ、DEF軽減後・クリティカル判定後のダメージへさらに固定 `x1.5`（`DamageFormula.DEFAULT_WEAKNESS_MULTIPLIER`）を乗算します。`weakness: NONE`（デフォルト）や、武器の `element` が一致しない場合は倍率なし。計算パイプライン全体は [`CombatDamageListener` 統一ダメージパイプライン](#combatdamagelistener-統一ダメージパイプライン)を参照してください。

## 通常モンスターの能動アビリティ（`MonsterAbilityCastService`）

以前はボス専用だった周期アビリティ発動が、通常モンスターにも `abilities`（`monsters.yml`）経由で使えるようになりました。ロジックは [Boss モジュールの `BossAbilityCastService`](boss.md) と全く同じパターンをそのまま複製したもので（回帰リスクを避けるため意図的にコード共有していません）、`AGGRO_RANGE=24.0` 内でクールダウン明けのアビリティを最大1つ、`tick()`（20tickごと）で発動します。`type: AOE_SLAM`（近くのプレイヤーへ直接ダメージ）と `FIREBALL_BARRAGE`（`SmallFireball` を発射、着弾判定は `BossFireballHitListener` を共有）に対応。

登録経路は「ボス湧きを経由しない湧き」に限定されます — `MonsterSpawnPointService#tick` と `AdminCommand`（`/oladmin spawn`）の実際のスポーン呼び出し箇所でのみ `registerIfAble` が呼ばれ、`MonsterSpawnService.spawn` 自体や `BossModule.spawn`（ボス湧き）では登録されません。そのため `bosses.yml` の `monster-id` と重複するモンスターIDでも `BossAbilityCastService` との二重詠唱は構造的に起きません。ダメージは `CombatDamageListener` を経由せず `player.damage(amount)` を直接呼ぶため、モンスターの通常攻撃力で上書きされません。

## `CombatDamageListener` 統一ダメージパイプライン

被ダメージイベントを処理するリスナーは、`monster`・`skill`・`status` の3モジュールにまたがっていた旧 `WeaponUseListener`/`CombatStatusListener`/`MonsterCombatListener` の3本立てから、**`rpg.monster.listener.CombatDamageListener`（`monster` モジュール側で登録） 1本に統一**されました。3本とも同じ `LOW` 優先度で動いていたため、Bukkitの同優先度リスナー間の実行順序が未定義であることに起因し、「クリティカルがATK%/DEFより先に乗る/後に乗る」といった順序不整合が起き得た問題を解消しています。

計算ロジック自体は Bukkit非依存の `rpg.status.combat.DamageFormula.compute(...)` に切り出されており、武器/素手/モンスター/スキルのすべてのダメージがこの1メソッドを通ります。順序は固定です:

```
1. base attack power（武器の baseAttackPower、素手ならATK、モンスターなら attack-power、スキルなら事前計算済みの値）
2. -> ATK%（applyAttackBonus: damage *= 1 + atkPercent/100）
3. -> DEF軽減（mitigate: damage *= 1 - defense/(defense+100)）
4. -> クリティカル判定・倍率（rollCrit → 命中なら damage *= baseCritMultiplier + critDmgPercent/100）
5. -> 属性弱点（applyElementalWeakness: 弱点一致なら damage *= 1.5）
```

`CombatDamageListener.onDamage`（`LOW` 優先度、`EntityDamageByEntityEvent`）は攻撃者ごとに base attack power・ATK%・クリティカル関連値を解決し（プレイヤーなら装備武器 or 素手、モンスターなら `MonsterData`）、被害者の防御力を解決し、`isWeaknessHit` で弱点一致を判定してから `DamageFormula.compute` を呼び出します。結果はScaled Health（上記）に応じてバニラ換算・スケール済み現在HP減算の両方に反映されます。

スキル発動によるダメージ（`rpg.skill.executor.SkillDamage`）は、攻撃者に `DamageFormula.SKILL_OVERRIDE_METADATA` メタデータを立てることで、`CombatDamageListener` に「base attack power + ATK%までは計算済み」と伝え、DEF/クリティカル/弱点だけを対象ごとに解決させます（AOE/コーン系スキルが複数対象に同じ基礎ダメージを一度だけ計算し、対象ごとの防御力差はそれぞれ反映するため）。

## その他のリスナー

- **`MonsterDeathListener`** — スポーン地点のスロットを解放。プレイヤーによる撃破ならバニラのドロップ/経験値をクリアし `rewardKiller` を呼ぶ
- **`VanillaHostileSpawnBlockerListener`** — `CreatureSpawnEvent` を理由 `NATURAL`/`SPAWNER` かつ `Monster` エンティティに限りキャンセル。`config.yml: monster.disable-vanilla-hostile-spawning`（デフォルト true）でゲート
- **`MonsterSunImmunityListener`** — タグ付きOreliaモンスターの日照による自然発火（`EntityCombustEvent` の基底クラスそのもの、溶岩/炎ブロックや他エンティティ由来のサブクラスは対象外）をキャンセルする

## 他モジュールへの依存

- **item** — `ElementType`, `ItemManager.createWeapon`（ドロップ用）、`WeaponIdentityService`（`CombatDamageListener` が武器の base attack power／クリティカル値を解決するため）
- **economy** — `deposit`
- **status** — `addExperience`、`StatusService.getFinalStats`（`CombatDamageListener` がプレイヤーのATK/DEF/CRT/CRT_DMGを読む）、`ScaledHealthService`
- **database** — スポーン地点の永続化
- **boss** — `BossAbilityCastService.FIREBALL_METADATA` を `MonsterAbilityCastService` が共有利用（火球着弾判定の重複実装を避けるため）
