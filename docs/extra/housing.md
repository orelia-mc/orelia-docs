# Housing モジュール（`rpg.extra.housing`）

設定駆動の購入可能な住居プロット。orelia-worldのダンジョン入口と同様、プレイヤーごとに生成されるインスタンスではなく、`housing.yml`で定義された固定のテレポート地点です。1プレイヤーにつき1軒まで所有できます。決済はVaultの`Economy`を直接使用します。

## ドメインモデル

```java
class HousePlot {
    String id, name;
    double price;
    String world;
    double x, y, z;
    float yaw;
}
```

## `housing.yml` の実際のキー

```yaml
plots:
  starter_1:
    name: "初心者向けの家 1番区画"
    price: 1000
    world: world
    x: 100
    y: 64
    z: 100
    yaw: 0
```

## `house_ownership` テーブル

```sql
CREATE TABLE IF NOT EXISTS house_ownership (
    owner_uuid VARCHAR(36) PRIMARY KEY,
    plot_id VARCHAR(64) NOT NULL,
    purchased_at BIGINT NOT NULL
)
```

## サービス/マネージャー

- **`HousePlotRepository`** — `load(YamlConfiguration)`, `findById`, `getAll()`（`housing.yml`をインメモリのテンプレートへパース、DBではない）
- **`HouseOwnershipRepository`**（`SchemaOwner`） — `loadAll()`（owner→plotIdの全件）, `save(ownerId, plotId)`, `findPlotByOwner(UUID)`
- **`HousingService`** — `ownerToPlot` をメモリキャッシュ（`loadAll()`で起動時ロード）。`ActionResult`（`OK`, `PLOT_NOT_FOUND`, `PLOT_TAKEN`, `ALREADY_OWN_A_HOUSE`, `INSUFFICIENT_FUNDS`, `NO_HOUSE`, `WORLD_NOT_FOUND`）を返す：
    - `purchase(player, plotId)` — 未所有・未売約・所持金十分を確認し`economy.withdrawPlayer`
    - `getOwnedPlot(UUID)` / `getAvailablePlots()`（既に売れた区画を除外）
    - `teleportHome(player)` — 所有区画のワールドが存在しなければ`WORLD_NOT_FOUND`

## コマンド

`HousingCommand` → `/ol house [list|buy <plotId>|home]`。`list`は購入可能な区画をID・名前・価格付きで表示。

## 消費する orelia-core / orelia-world API

`DatabaseManager`・Vaultの`Economy`（いずれも`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）。`rpg.api`は使用しない。
