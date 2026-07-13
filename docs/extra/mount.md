# Mount モジュール（`rpg.extra.mount`）

設定駆動の騎乗マウント。Petモジュールとほぼ同じ構造（config駆動テンプレート＋DB永続化されたunlock/selection＋インメモリの召喚状態）で、召喚したエンティティにプレイヤーを騎乗させ、移動速度属性を`mounts.yml`の`speed`で上書きします。決済はVaultの`Economy`を直接使用します。

## ドメインモデル

```java
class MountDefinition {
    String id, name;
    EntityType entityType;
    double speed, price;
}
```

## `mounts.yml` の実際のキー

```yaml
mounts:
  horse:
    name: "馬"
    entity-type: HORSE
    speed: 0.225
    price: 1000
```

## `mount_unlock` / `mount_selection` テーブル

```sql
CREATE TABLE IF NOT EXISTS mount_unlock (
    owner_uuid VARCHAR(36) NOT NULL,
    mount_id VARCHAR(64) NOT NULL,
    unlocked_at BIGINT NOT NULL,
    PRIMARY KEY (owner_uuid, mount_id)
)
CREATE TABLE IF NOT EXISTS mount_selection (
    owner_uuid VARCHAR(36) PRIMARY KEY,
    mount_id VARCHAR(64) NOT NULL
)
```

## サービス/マネージャー

- **`MountConfigRepository`** — `load(YamlConfiguration)`, `findById`, `getAll()`（`mounts.yml`をインメモリのテンプレートへパース）
- **`MountOwnershipRepository`**（`SchemaOwner`） — `loadUnlocks()`, `loadSelections()`, `saveUnlock`, `saveSelection`（Petと同じ形）
- **`MountManager`** — 召喚中のエンティティ（`Entity`）をowner単位でインメモリ管理。`register`, `despawn`, `hasActiveMount`, `isTrackedMount(Entity)`（降車判定用）, `despawnAll()`
- **`MountService`** — `unlockedByOwner`/`selectedByOwner`をメモリキャッシュ。`ActionResult`（`OK`, `MOUNT_NOT_FOUND`, `ALREADY_UNLOCKED`, `NOT_UNLOCKED`, `INSUFFICIENT_FUNDS`, `NO_ACTIVE_MOUNT`）を返す：`unlock`, `summon`（エンティティをスポーンし`Attribute.GENERIC_MOVEMENT_SPEED`を`speed`で上書き、`entity.addPassenger(player)`で騎乗させる）, `dismiss`, `getAllMounts`/`getUnlockedMounts`

## コマンド

`MountCommand` → `/ol mount [list|buy <id>|summon [id]|dismiss]`。`summon`に引数が無ければ最後に選択したマウントを再召喚する。

## リスナー

`MountLifecycleListener` — `EntityDismountEvent`（プレイヤーが追跡中のマウントから降りたら即座にdespawn）と`PlayerQuitEvent`（切断時にdespawn）の2つを処理。

## 消費する orelia-core / orelia-world API

`DatabaseManager`・Vaultの`Economy`（いずれも`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）。`rpg.api`は使用しない。
