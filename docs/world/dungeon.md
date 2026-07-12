# Dungeon モジュール（`rpg.dungeon`）

パーティ人数制限付きのインスタンス風ダンジョン攻略（ワールドを複製するのではなく、固定の物理エリアへのテレポート入退場）と、クリア報酬。

## ドメインモデル

```java
class DungeonData {
    String id, name;
    DungeonType type;
    int minPartySize, maxPartySize;
    String world; double x, y, z;
    long rewardExp;
    double rewardMoney;
}

enum DungeonType { NORMAL, STORY, RAID, SOLO, PARTY }

class DungeonInstance {
    UUID id; // ランダム生成
    DungeonData data;
    Map<UUID, Location> membersAndReturnLocations;
    long startedAtMillis;
    volatile DungeonInstanceStatus status;
}

enum DungeonInstanceStatus { ACTIVE, COMPLETED, FAILED }
```

## `dungeons.yml` の実際のキー

```yaml
dungeons:
  raid_dragons_lair:
    name: "竜の巣窟"
    type: RAID
    min-party-size: 5
    max-party-size: 10
    world: world
    x: 300
    y: 70
    z: 300
    reward-exp: 1000
    reward-money: 500
```

## サービス

- **`DungeonRepository`** — `load`/`findById`/`getAll`
- **`DungeonInstanceManager`** — `register`, `getByPlayer(UUID)`, `remove(instanceId)`, `removePlayer(UUID)`
- **`DungeonService`**
    - `start(dungeonId, List<Player> party)` → `Optional<StartFailure>`（`UNKNOWN_DUNGEON`, `PARTY_TOO_SMALL`, `PARTY_TOO_LARGE`, `WORLD_NOT_FOUND`, `ALREADY_IN_DUNGEON`）。パーティを入口へテレポートし、復帰地点を記録
    - `finish(playerId, boolean success)` — パーティ全員を復帰地点へテレポートし、成功時は全員に `StatusApi.addExperience` でEXP、Vault `Economy` でお金を付与、インスタンスを削除

## リスナー

`DungeonQuitListener`（`PlayerQuitEvent`） — `instanceManager.removePlayer` を呼ぶ（マッピングのみ解放。インスタンスや他メンバーには影響しない）。

!!! note "プレイヤー向けコマンドは無い"
    このパッケージにはダンジョンの開始/終了を操作する `command/` クラスがありません（外部/将来の統合で駆動される想定）。`QuestProgressService.onDungeonCleared` はフックとして存在しますが、まだ `DungeonService` へ配線されていません。

## 消費する orelia-core API

`StatusApi`、加えて Vault `Economy`。
