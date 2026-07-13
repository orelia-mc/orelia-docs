# Achievement モジュール（`rpg.extra.achievement`）

設定駆動の実績。オンラインプレイヤー全員を定期的に全実績の条件と照合し、初回達成時にスキルポイントを付与します。orelia-extraの中で最も広く他プラグインへ依存するモジュールで、`orelia-world`へのソフト依存（`QuestApi`）を持つのはこのモジュールだけです。

## ドメインモデル

```java
class AchievementDefinition {
    enum ConditionType { REACH_LEVEL, COMPLETE_QUEST, MONEY_BALANCE }

    String id, name, description;
    ConditionType conditionType;
    String conditionValue; // レベル数値・クエストID・所持金額のいずれかを文字列で保持
    int rewardSkillPoints;
}
```

## `achievements.yml` の実際のキー

```yaml
achievements:
  reach_level_10:
    name: "駆け出し冒険者"
    description: "レベル10に到達する"
    condition-type: REACH_LEVEL
    condition-value: "10"
    reward-skill-points: 1
  goblin_slayer:
    name: "ゴブリン退治人"
    description: "「king_of_goblins」クエストを完了する（要orelia-world）"
    condition-type: COMPLETE_QUEST
    condition-value: "king_of_goblins"
    reward-skill-points: 3
```

## `achievement_unlock` テーブル

```sql
CREATE TABLE IF NOT EXISTS achievement_unlock (
    owner_uuid VARCHAR(36) NOT NULL,
    achievement_id VARCHAR(64) NOT NULL,
    unlocked_at BIGINT NOT NULL,
    PRIMARY KEY (owner_uuid, achievement_id)
)
```

## サービス/マネージャー

- **`AchievementConfigRepository`** — `load(YamlConfiguration)`, `getAll()`（`achievements.yml`をインメモリのテンプレートへパース）
- **`AchievementProgressRepository`**（`SchemaOwner`） — `loadAll()`（owner→解除済みachievementIdの集合）, `saveUnlock`
- **`AchievementService`** — `unlockedByOwner`をメモリキャッシュ（`loadAll()`で起動時ロード）。
    - `checkAll()` — 全オンラインプレイヤーに対し`checkPlayer`を実行。`AchievementModule`が`SchedulerService.runTimer`で600tick（30秒）ごとに呼び出す
    - `checkPlayer(player)` — 未解除の実績ごとに`isMet`を判定し、達成していれば解除・永続化・`grantReward`
    - `isMet` — `REACH_LEVEL`は`statusApi.getLevel`、`COMPLETE_QUEST`は`questApi != null && questApi.hasCompletedQuest(...)`（orelia-world未導入なら常にfalse）、`MONEY_BALANCE`は`economy != null && economy.getBalance(player) >= ...`
    - `grantReward` — `rewardSkillPoints > 0`なら`skillApi.grantSkillPoints`、プレイヤーへ実績解除メッセージを送信

## コマンド

`AchievementCommand` → `/ol achievement` で全実績を`[済]`/`[未]`マーク付きで一覧表示。

## リスナー

`AchievementJoinListener`（`PlayerJoinEvent`） — 参加直後に`checkPlayer`を実行し、次の定期スイープを待たずに実績判定する。

## 消費する orelia-core / orelia-world API

- `StatusApi`, `SkillApi`（いずれも`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）
- Vaultの`Economy`（ソフト — `null`許容、`MONEY_BALANCE`条件が判定不能になるだけ）
- `rpg.world.api.QuestApi`（ソフト — orelia-worldが未導入なら`null`、`COMPLETE_QUEST`条件が常にfalseになるだけ）
