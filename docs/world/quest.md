# Quest モジュール（`rpg.quest`）

Config駆動のクエスト定義と、受注 → 進行 → 報告 → 報酬付与の完全な状態機械。`orelia-core` のstatus/item/skill/combat APIの上に乗る、コンテンツ層の主役です。

## ドメインモデル

```java
class QuestData {
    String id, name;
    QuestType type;
    List<String> description;
    List<QuestObjective> objectives;
    QuestReward reward;
    boolean repeatable, partyOnly;
    int requiredLevel;
    List<String> prerequisiteQuestIds;
    int availableHourStart, availableHourEnd; // -1 で時間帯ゲート無効

    boolean isAvailableAtHour(int hour);
}

class QuestObjective {
    ObjectiveType type;
    String targetId;
    int requiredAmount;
    String world;
    double x, y, z, radius;
}

class QuestReward {
    long exp;
    double money;
    String weaponId, accessoryId;
    int skillPoints;
    String title;
    String vanillaMaterial;
    int vanillaAmount;
}

enum ObjectiveType { KILL_MONSTER, COLLECT_ITEM, DELIVER_ITEM, REACH_LOCATION, TALK_NPC, KILL_BOSS, CLEAR_DUNGEON }
enum QuestType { MAIN, SUB, DAILY, WEEKLY, EVENT }
enum QuestState { NOT_ACCEPTED, IN_PROGRESS, ACHIEVED, AWAITING_REPORT, COMPLETE }

class PlayerQuestComponent implements PlayerDataComponent {
    Map<String, PlayerQuestProgress> activeQuests;
    Set<String> completedQuestIds;
    Set<String> titles;

    void startQuest(...); void completeQuest(...); boolean hasCompleted(...); void addTitle(...);
}

class PlayerQuestProgress {
    QuestState state;
    Map<Integer, Integer> objectiveProgress; // 目標インデックスごとの進捗
}
```

## `quests.yml` の実際のキー

```yaml
quests:
  slime_cull:
    name: "スライム討伐"
    type: MAIN
    required-level: 1
    repeatable: false
    party-only: false
    prerequisite-quests: []
    objectives:
      obj_1:
        type: KILL_MONSTER
        target-id: forest_slime
        amount: 5
    reward:
      exp: 50
      money: 20
      skill-points: 1
```

## サービス/マネージャー

- **`QuestRepository`** — `load(YamlConfiguration)`, `findById`, `getAll()`
- **`PlayerQuestRepository`**（`SchemaOwner`） — テーブル `quest_completed`, `quest_title`, `quest_progress`。`loadOrCreate(uuid)`, `save(component)`
- **`QuestManager`**（`PlayerDataComponentLoader<PlayerQuestComponent>`） — repository と `PlayerDataManager.registerLoader` を橋渡し
- **`QuestEligibilityService.checkEligibility(Player, QuestData)`** → `Optional<Ineligibility>`（`ALREADY_ACTIVE`, `ALREADY_COMPLETED`, `LEVEL_TOO_LOW`, `PREREQUISITE_MISSING`, `NOT_AVAILABLE_NOW`）
- **`QuestItemInventoryService`** — `countMatching`, `consume`。武器IDは `ItemApi.identifyWeapon` 経由、それ以外はバニラ `Material` でマッチング
- **`QuestProgressService`** — `accept`, `report`, `checkPeriodicObjectives`（オンラインプレイヤーに対しスケジューラのタイマーで実行、`config.yml: quest.objective-check-period-ticks`＝40tick周期）, `onMonsterKilled`, `onBossKilled`, `onNpcTalked`, `onDungeonCleared`（フックのみ、まだ未配線）, `setObjectiveProgress`
- **`QuestRewardService.grant(Player, QuestReward)`** — `StatusApi` 経由でEXP付与、Vault `Economy` 経由でお金付与、`ItemApi.createWeapon` で武器、`AccessoryApi.createAccessory` で装飾品、`SkillApi.grantSkillPoints` でスキルポイント、称号を `PlayerQuestComponent` へ追加、バニラアイテムを付与/ドロップ

## コマンド

`QuestCommand` → `/rpgquest list`（進行中クエストと状態を表示）、`/rpgquest abandon <id>`（報酬無しで放棄）

## GUI

`QuestGuiScreen.build(Player, List<String> offeredQuestIds)` — orelia-coreの `Gui`/`GuiButton`/`ItemBuilder` を使い27スロットの「クエスト」画面を構築。クリックは `report`/`accept` へルーティング。

## リスナー

`QuestKillListener`（`EntityDeathEvent`） — `CombatApi.identifyMonster`/`identifyBoss` を使い KILL_MONSTER/KILL_BOSS 目標を進行させる。

## 消費する orelia-core API

`StatusApi`, `ItemApi`, `AccessoryApi`, `SkillApi`, `CombatApi`、加えて `DatabaseManager` と Vault `Economy`（ソフト依存、null許容）。
