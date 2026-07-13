# Job モジュール（`rpg.job`）

静的でconfig駆動の職業/クラスシステム（剣士/槍士/戦士/弓士/釣り人）。武器種を制限し、パッシブなステータス補正を与えます。経験値やレベリングは持たず、変更するまで維持されるフラットな状態です。

## ドメインモデル

```java
enum JobType { SWORDSMAN, SPEARMAN, WARRIOR, ARCHER, FISHERMAN } // 固定。追加には実装変更が必要

class Job {
    JobType type;
    String displayName;
    Set<WeaponType> allowedWeapons;
    StatSheet passiveBonus;

    boolean canUse(WeaponType type);
}

class PlayerJobComponent implements PlayerDataComponent {
    UUID owner;
    JobType currentJob; // null = 未就業
}
```

## `jobs.yml` の実際のキー

```yaml
jobs:
  SWORDSMAN:
    display-name: "剣士"
    allowed-weapons: [SWORD]
    passive-bonus:
      ATK: 2
      AGI: 1
```

!!! note "FISHERMAN はまだステータス補正なし"
    `FISHERMAN`（釣り人）はジョブとしてのみ追加済みです。レベリングやパッシブ補正の設計は未定のため、`jobs.yml` の該当セクションは `allowed-weapons: []` のみで `passive-bonus` を持ちません。

## サービス/マネージャー

- **`JobManager`**（`PlayerDataComponentLoader`） — `setDefinitions`, `getDefinition`, `getDefinitions`, `loadOrCreate`, `save`
- **`JobService`**
    - `getCurrentJob(UUID)`
    - `changeJob(UUID, JobType)` — 定義が無ければ false。あれば `currentJob` を設定し、`statusService.setEquipmentContribution(uuid, "job", passiveBonus)` を呼ぶ。`changeJob` 自体には要件ゲートは無い
    - `canUseWeaponType(UUID, WeaponType)`
- **`JobConfigLoader.load(YamlConfiguration)`**
- **`PlayerJobRepository`**（`SchemaOwner`） — テーブル `player_job(uuid, job)`

## コマンド

`JobCommand` → `/ol job`

- `job list` — 全 `JobType` を表示
- 引数なし — 現在の職業、または「まだ選択されていません。NPCを訪ねてください」的なメッセージ

!!! note "職業変更コマンドは無い"
    職業変更は（`orelia-world` の）NPC経由でのみ行われます。このコマンドでは変更できません。

## 他モジュールへの依存

- **item** — `WeaponType`（許可武器の判定）
- **status** — `StatSheet`/`StatType` および `StatusService.setEquipmentContribution`（パッシブ補正、ソースキー `"job"`）
