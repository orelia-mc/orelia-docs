# Gathering モジュール（`rpg.gathering`）

採掘・木こり（ブロック採取）と農業（作物の一括植え付け・収穫）、および両者が共有するレベル/範囲拡張システムを持つモジュール。`DatabaseModule` のみに依存する独立系統で、レベル・経験値は Status のキャラクターレベルとは別管理です。

## ドメインモデル

```java
enum GatherActionType { MINING, WOODCUTTING }

record GatherBlockTemplate(
    Material blockType, GatherActionType actionType, int cooldownSeconds,
    Material replaceBlock, int xpGain, int minLevel) {}

record CropTemplate(Material cropType, Material seedItem, int xpGain) {}

record LevelRange(int minLevel, int maxLevel, int radius) {}

record BlockRegenTask(UUID id, String world, int x, int y, int z,
    String originalMaterial, long restoreAtMillis) {}

class PlayerGatheringComponent implements PlayerDataComponent {
    UUID owner;
    int level;      // 1始まり
    long experience;
}
```

## `gathering.yml` の実際のキー

```yaml
regen-tick-period-ticks: 100 # 100 ticks = 5秒

gather-settings:
  MINING:
    DIAMOND_ORE:
      cooldown-seconds: 300
      replace-block: BEDROCK
      xp-gain: 50
      min-level: 0
  WOODCUTTING:
    OAK_LOG:
      cooldown-seconds: 60
      replace-block: AIR
      xp-gain: 10
      min-level: 0

farm-settings:
  WHEAT:
    seed-item: WHEAT_SEEDS
    xp-gain: 8

leveling:
  mode: FORMULA # FORMULA または TABLE
  formula:
    a: 50
    b: 1.5
    c: 100
  table: []
  max-level: 50

level-ranges:
  - min-level: 1
    max-level: 9
    radius: 0 # 1x1（バニラ）
  - min-level: 40
    max-level: 50
    radius: 4 # 9x9
```

## 採取（採掘・木こり） — SOW 3.1

`GatherBlockBreakListener` が `BlockBreakEvent` をフックします。

- 対象ブロック（`gather-settings` に定義されたもの）を破壊すると、常に `replace-block` へ一時的に置き換わり、`cooldown-seconds` 経過後に元のブロックへ復元されます。
- イベントをキャンセルしていないため、バニラのドロップ処理はそのまま走ります。置き換え自体は **次のtick** にずらして実行します（`BlockRegenService#scheduleNextTick`）。バニラのブロック除去処理はリスナーの return 後に走るため、同一tick内で置き換えるとレースして誤ったブロックが破壊されてしまうためです。
- スニーク中はプレイヤーのレベルに応じた半径（立方体、`level-ranges` 参照）内の同一ブロックも `Block#breakNaturally` でまとめて採取し、それぞれに同じ復元タスクを積みます（こちらは同期的に即時破壊されるためレースなし）。
- `min-level` を満たさないプレイヤーはイベントがキャンセルされます。
- WorldGuard で保護された座標は `RegionProtectionService`（後述）でスキップされます。

### サーバー再起動時のデータ保護

`BlockRegenRepository`（`SchemaOwner`）がテーブル `block_regen_task(id, world, x, y, z, original_material, restore_at_millis)` を持ち、破壊のたびに非同期で保存します。`GatheringModule#onEnable` で `BlockRegenService#loadPending` が全件を読み込み直し、`regen-tick-period-ticks` ごとのメインスレッドtickでチャンクがロードされていれば復元します。チャンクがアンロード中に期限が来た場合は `GatherChunkLoadListener`（`ChunkLoadEvent`）が該当チャンクのタスクだけを即座にチェックします。

## 農業 — SOW 3.2

`FarmingListener` が植え付け・収穫を担当します。

- **植え付け**: `FARMLAND` を右クリック（種アイテムを保持）した通常時はバニラのまま。スニーク時は半径内（平面）の空いている耕地へ一括植え付けし、手持ちの種の個数を上限に1個ずつ消費します。
- **収穫**: 成長しきった作物を破壊すると常に経験値が入ります。スニーク＋クワを持っている場合のみ、半径内の成長しきった同種作物を一括収穫し、クワの残り耐久値を上限に1ダメージずつ消費します（耐久が尽きたら道具は消滅）。
- 収穫後の自動再植え付けは行いません（手動植え付けの原則を維持）。

## レベル・範囲拡張システム — SOW 3.3

採掘・木こり・農業のすべての経験値が同じ `PlayerGatheringComponent` に加算されます。

- **`GatheringLevelingConfig`** — `leveling.mode` が `FORMULA` なら `NextXP = a * level^b + c`、`TABLE` なら `leveling.table` の該当インデックス（配列末尾を超えたレベルは最後の値を再利用）。`leveling.max-level` に到達すると経験値は増えなくなります。レベル上限を50より先に拡張する場合は `max-level`（と必要なら `table`/`level-ranges`）を書き換えるだけで済み、コード変更は不要です。
- **`LevelRadiusConfig`** — `level-ranges` の各バンドに `radius`（0=1x1, 1=3x3, ...）を対応付け。設定済みバンドの最大レベルを超えたレベルは最後のバンドの `radius` を引き継ぎます。
- 一括処理はすべて **スニーク時のみ** 発動します（誤操作防止、SOW 3.3）。

## サービス/マネージャー

- **`GatheringManager`**（`PlayerDataComponentLoader`） — `PlayerGatheringRepository` 経由でレベル/経験値をロード・保存
- **`GatheringLevelService`** — `addExperience(UUID, long)`（レベルアップ処理込み）, `getLevel(UUID)`
- **`BlockRegenService`** — `schedule`/`scheduleNextTick`（置き換え＋タスク登録）, `loadPending`, `start`, `onChunkLoaded`
- **`RegionProtectionService`** — WorldGuard 連携（後述）
- **`GatheringDefinitionRepository`** — `gathering.yml` の `gather-settings`/`farm-settings` をパース

## WorldGuard 連携

`build.gradle.kts` に WorldGuard へのコンパイル時依存は **追加していません**（WorldGuard の Maven リポジトリはこの構成のビルド環境からアクセスできない前提のため）。代わりに `RegionProtectionService` がリフレクションで `com.sk89q.worldguard.bukkit.WorldGuardPlugin#canBuild(Player, Block)` を呼び出します。WorldGuard 未導入時、またはAPI形状が想定と異なる場合はすべて許可（バニラと同じ挙動）にフォールバックします。`plugin.yml` には `softdepend: [WorldGuard]` のみ追加しています。

## コマンド

`GatheringCommand` → `/ol gathering` — 送信者の採取/農業レベルと現在の一括処理半径を表示。

## 他モジュールへの依存

- **database** — `PlayerGatheringRepository`/`BlockRegenRepository` はどちらも `DatabaseManager` 経由の `SchemaOwner`
- 他モジュールへの依存はこれのみ（Status のキャラクターレベルとは独立）
