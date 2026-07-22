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
    # abilities は設定しない — bosses.yml の monster-id からも参照されるモンスターに abilities を
    # 設定すると、ボスとして湧いたときに BossAbilityCastService と衝突しうるため意図的に避ける
```

`weakness` は省略時 `NONE`（弱点なし）。`crit-rate`（省略時0=常時非クリティカル）・`crit-multiplier`（省略時1.5、`crit-rate > 0` のときのみ意味を持つ）も任意キーとして追加できます。`abilities` は `bosses.yml` の `abilities` ブロックと同じ形（`type: AOE_SLAM | FIREBALL_BARRAGE`）です。`monsters.yml` のコメントは、`bosses.yml` の `monster-id` として参照されているモンスターID（`goblin_king`, `flame_lord`）に `abilities` を設定しないよう明記しています——`MonsterAbilityCastService` への登録は `BossModule.spawn()` を経由しない湧き経路（スポーンポイント、`/oladmin monster spawn`）でのみ行われるため実害はありませんが、意図が分かりにくくなるための運用上の注意です。

## サービス

- **`MonsterRepository`** — `load`, `findById`, `getAll`
- **`MonsterSpawnService`**
    - `spawn(String monsterId, Location)` / `spawn(..., spawnPointId)` — バニラエンティティをスポーンし、PDCタグ `monster_id`/`spawn_point_id` を付与。`Attribute.MAX_HEALTH` を設定して `setHealth`、`mob.setAware(aiType != PASSIVE)`。攻撃力/防御力はバニラ属性ではなく**リスナー内で被弾ごとに適用**
    - `idOf`, `spawnPointIdOf`, `dataOf`
- **`MonsterDropService(ItemManager, EconomyService, StatusService).rewardKiller(data, killer, location)`** — `statusService.addExperience`、`economyService.deposit(MathUtil.lerp(moneyMin, moneyMax, random))`、各 `DropEntry` を `MathUtil.rollChance` で判定し `itemManager.createWeapon` またはバニラ素材で自然ドロップ
- **`MonsterKeys`** — PDCキー `monster_id`, `spawn_point_id`
- **`MonsterSpawnPointService`** — `loadAll()`, `add(Player admin, monsterId, intervalSeconds, maxAlive)`, `remove(UUID)`, `getAll()`, `tick()`（20tickごとにチェック） — `aliveCount >= maxAlive` または未到来ならスキップ、それ以外はスポーンして記録。`onEntityRemoved` でスロットを解放
- **`MonsterSpawnPointManager`** — 実行時の帳簿管理。`isDueToSpawn` = `now - lastSpawn >= intervalSeconds * 1000`
- **`MonsterSpawnPointRepository`**（`SchemaOwner`） — テーブル `monster_spawn_point(id, monster_id, world, x, y, z, interval_seconds, max_alive)`

## リスナー

- **`MonsterCombatListener`**（`LOW` 優先度） — ダメージ量を `attackPower` に上書き。被害者がモンスターなら `defense/(defense+100)` の軽減を適用
- **`MonsterDeathListener`** — スポーン地点のスロットを解放。プレイヤーによる撃破ならバニラのドロップ/経験値をクリアし `rewardKiller` を呼ぶ
- **`VanillaHostileSpawnBlockerListener`** — `CreatureSpawnEvent` を理由 `NATURAL`/`SPAWNER` かつ `Monster` エンティティに限りキャンセル。`config.yml: monster.disable-vanilla-hostile-spawning`（デフォルト true）でゲート

## 他モジュールへの依存

- **item** — `ElementType`, `ItemManager.createWeapon`（ドロップ用）
- **economy** — `deposit`
- **status** — `addExperience`
- **database** — スポーン地点の永続化
- **boss** — Javadoc上の参照のみ（`skillIds` は将来のフック）
