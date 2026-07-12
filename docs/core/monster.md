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
    AiType aiType;
    List<DropEntry> drops;
    long expReward;
    double moneyMin, moneyMax;
    List<String> skillIds;
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
    skills: []
```

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
