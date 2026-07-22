# Status モジュール（`rpg.status`）

レベル/経験値の進行と、派生する戦闘ステータス（HP, SP, ATK, DEF, CRT, CRT_DMG, SPD の7種。AGI/DEX/INTは削除済み）、およびHP/SP自然回復を管理します。他の全モジュールは独自に計算せず、`StatusService` 経由で有効ステータスを読みます。**登録順2番目（Databaseの次）の、他モジュールに依存しない基盤モジュール**です。

実際のダメージ計算パイプライン（ATK% → DEF軽減 → クリティカル → 属性弱点）は `rpg.status.combat.DamageFormula`（純粋な計算ロジック）と、それを呼び出す `rpg.monster.listener.CombatDamageListener`（唯一のBukkitイベントリスナー、`monster` モジュール側で登録）に一本化されています。詳細は [Monster モジュール仕様](monster.md#combatdamagelistener-統一ダメージパイプライン) を参照してください。`status` モジュール自身はATK/DEF/CRT/CRT_DMGの値をステータスとして計算・提供するだけで、ダメージイベント自体は処理しません（旧 `CombatStatusListener` は撤去済み）。

## ドメインモデル

```java
enum StatType { HP, SP, ATK, DEF, CRT, CRT_DMG, SPD } // AGI/DEX/INTは削除済み

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

## Scaled Health（プレイヤーの体力スケーリング）

プレイヤーの実際の「体力プール」は `StatType.HP`（レベル・装備・バフで数百〜に達し得る）ですが、バニラの `Attribute.MAX_HEALTH` は小さい安全な範囲でしか扱えません。`rpg.status.service.ScaledHealthService`（Bukkitに依存しない純粋ユーティリティ）が、この「スケール済みHP」とバニラ体力を常に同じ割合に保ちます。

- **`ScaledHealthService.syncVanillaHealth(entity, scaledCurrent, scaledMax)`** — バニラ体力を `scaledCurrent/scaledMax` と同じ割合に設定（`entity.isDead()` なら何もしない — 死亡演出中に `setHealth` を呼ぶと片死に状態になる不具合の回避）。
- **`ScaledHealthService.convertDamageToVanilla(victim, scaledDamage, scaledMax)`** — スケール済みダメージ量をバニラ体力の同割合の量に変換し、`EntityDamageEvent#setDamage` にそのまま渡せるようにする（ノックバック・被弾音・死亡処理などBukkit標準の解決に任せるため）。

`StatusService` 側の対応するメソッド：

- **`applyScaledCombatDamage(UUID, scaledAmount)`** — `CombatDamageListener` が計算済みのスケール済みダメージで `currentHp` のみを減算（バニラ体力は `CombatDamageListener` が変換値を `event.setDamage` に渡すことで別途更新される）。
- **`applyEnvironmentalDamage(UUID, vanillaAmount)`** — 落下/落雷/溺れなど `CombatDamageListener` を通らないダメージ（`ScaledHealthEnvironmentalDamageListener` から呼ばれる）は、実際にバニラ体力へ加わった割合を `currentHp` 側にもミラーする。
- **`damage`/`heal`/`tickRegen`/`addExperience`** はいずれも変更後に `syncVanillaHealth` を呼び、バニラ体力を追従させる。

`ScaledHealthJoinListener` が参加時にバニラ体力を初期同期し、`ScaledHealthRegenListener` が自然回復のたびに同期します。モンスター/ボス側の同種の仕組みは [Monster モジュール仕様](monster.md#scaled-health-モンスターの体力スケーリング) を参照してください。

## 永続化

`StatusRepository` は `level, experience, current_hp, current_sp` のみを永続化します。装備コントリビューション/バフは実行時のみの状態で、join時に item/accessory/skill/job 各モジュールが再構築します。

## 依存関係

`status` は他モジュールへの依存を持たない葉（基盤）モジュールです。逆に、item, job, skill, accessory, monster, gui, api から参照されます。ダメージ計算のリスナー自体（`CombatDamageListener`）は `monster` モジュール側に置かれ、`status` の `StatusService`/`ScaledHealthService` を呼び出す形になっている点に注意してください（`status` → `monster` への依存はありません。逆方向の依存です）。
