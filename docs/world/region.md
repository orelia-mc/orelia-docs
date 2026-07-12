# Region モジュール（`rpg.world.region`）

名前付きの軸並行境界ボックスエリア（町/フィールド/ダンジョン入口/ワープ）と、入退場チャットメッセージ、自動ワープテレポート。

## ドメインモデル

```java
class RegionData {
    String id, name;
    RegionType type;
    String world;
    // コンストラクタで min/max の x/y/z を自動正規化
    double minX, minY, minZ, maxX, maxY, maxZ;
    String enterMessage, exitMessage;
    String warpWorld; double warpX, warpY, warpZ, warpYaw;

    boolean contains(String worldName, double x, double y, double z);
}

enum RegionType { AREA, TOWN, FIELD, DUNGEON_ENTRANCE, WARP }
```

## `regions.yml` の実際のキー

```yaml
regions:
  goblin_cave_entrance:
    name: "ゴブリンの洞窟入口"
    type: WARP
    world: world
    bounds: { min-x: 95, min-y: 60, min-z: 95, max-x: 105, max-y: 70, max-z: 105 }
    enter-message: "&eゴブリンの洞窟へ転送します..."
    warp: { world: world, x: 100, y: 64, z: 100, yaw: 0 }
```

## サービス

- **`RegionRepository`** — `load`/`findById`/`getAll`
- **`RegionTracker`** — `Map<UUID, String> currentRegion`。プレイヤーごとの重複入退場発火を防ぐ
- **`RegionService`**
    - `findRegionAt(Location)`（線形走査、空間インデックス無し — 「手作業で設定する規模なら十分」との設計判断）
    - `warp(Player, RegionData)`（`WARP` タイプのみ発火）

## リスナー

`RegionMoveListener`（`PlayerMoveEvent`、ブロック境界を跨いだ時のみ発火。`PlayerQuitEvent` でトラッカーをクリア） — 退場/入場メッセージを送信し、ワープを発火。

!!! info "このモジュールだけの特徴"
    このリストの中で唯一 `rpg.api.*` にも `DatabaseManager` にも依存しません。純粋な Bukkit `Location`/`World` 操作のみで完結し、リージョンは一時的な実行時状態のみを持ちます（プレイヤーごとの永続化は無し）。
