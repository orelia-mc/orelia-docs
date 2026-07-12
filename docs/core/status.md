# Status モジュール（`rpg.status`）

レベル/経験値の進行と、派生する戦闘ステータス（HP, SP, ATK, DEF, AGI, DEX, INT, CRT, CRT_DMG, SPD）、およびHP/SP自然回復を管理します。他の全モジュールは独自に計算せず、`StatusService` 経由で有効ステータスを読みます。**登録順2番目（Databaseの次）の、他モジュールに依存しない基盤モジュール**です。

## ドメインモデル

```java
enum StatType { HP, SP, ATK, DEF, AGI, DEX, INT, CRT, CRT_DMG, SPD }

class StatSheet {
    Map<StatType, Double> values;
    void set(StatType, double);
    double get(StatType);
    StatSheet plus(StatSheet other); // 合算マージ
    StatSheet copy();
    Map<StatType, Double> asMap();
    static StatSheet empty();
}

enum ModifierType { FLAT, PERCENT }

class StatModifier {
    UUID id;
    String sourceKey;
    StatType statType;
    ModifierType modifierType;
    double amount;
    long expiresAtMillis;
    boolean isExpired(long now);
}

class PlayerStatusComponent implements PlayerDataComponent {
    UUID owner;
    int level;
    long experience;
    StatSheet baseStats;
    Map<String, StatSheet> equipmentContributions;
    List<StatModifier> buffs;
    double currentHp, currentSp;
}

record LeaderboardEntry(UUID uuid, String name, int level, long experience) {}
```

## `config.yml` の該当キー

```yaml
status:
  regen:
    hp-percent-per-tick: 0.5
    sp-percent-per-tick: 1.0
    period-ticks: 100
  leveling:
    exp-per-level: 100
    max-level: 100
  growth:
    HP:  { base: 100, per-level: 10 }
    SP:  { base: 50,  per-level: 5 }
    ATK: { base: 5,   per-level: 1 }
    # StatTypeごとに1ブロック
```

## 計算式

- **`LevelingConfig.requiredExperience(level)`** = `expPerLevel * level`（累積ではなく線形）
- **`LevelGrowthService.baseStatsForLevel(level)`** — 各ステータスにつき `base + perLevel * max(0, level - 1)`
- **`StatusCalculatorService.calculateFinal`** — base + 全装備コントリビューションを `plus` で合算 → バフ適用は「フラット優先、その後パーセント」：

    ```
    finalValue = flatTotal * (1 + percentTotal / 100.0)  // 0未満にはクランプ
    ```

- **`StatusService.addExperience(UUID, long)`** — `while level < max && exp >= required(level)` でレベルアップをループ処理。レベルアップごとに baseStats を再計算し、現在HP/SPを新しい最大値まで回復。
- **`tickRegen(UUID, hpPercent, spPercent)`** — `currentHp += maxHp * pct / 100`（クランプ）。毎tick期限切れバフを剪定。

その他: `setEquipmentContribution`, `clearEquipmentContribution`, `addBuff`, `removeBuffsFromSource`, `tryConsumeSp`, `damage`, `heal`, `getLeaderboard`。

## 戦闘ダメージ計算（`CombatStatusListener`）

`EntityDamageByEntityEvent` を処理：

```
damage *= 1 + attackerATK / 100
reduction = victimDEF / (victimDEF + 100)
damage *= (1 - reduction)
```

## 永続化

`StatusRepository` は `level, experience, current_hp, current_sp` のみを永続化します。装備コントリビューション/バフは実行時のみの状態で、join時に item/accessory/skill/job 各モジュールが再構築します。

## 依存関係

`status` は他モジュールへの依存を持たない葉（基盤）モジュールです。逆に、item, job, skill, accessory, monster, gui, api から参照されます。
