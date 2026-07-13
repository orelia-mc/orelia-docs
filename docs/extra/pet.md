# Pet モジュール（`rpg.extra.pet`）

設定駆動の追従ペット。プレイヤーは`pets.yml`で定義されたペットを購入（unlock）し、召喚（summon）・解除（dismiss）できます。召喚中のエンティティ自体はインメモリのみで、サーバー再起動をまたいで消えます（unlock/selectionの記録は永続化されます）。決済はVaultの`Economy`を直接使用します。

## ドメインモデル

```java
class PetDefinition {
    String id, name;
    EntityType entityType;
    double price;
}
```

## `pets.yml` の実際のキー

```yaml
pets:
  wolf:
    name: "オオカミ"
    entity-type: WOLF
    price: 500
```

## `pet_unlock` / `pet_selection` テーブル

```sql
CREATE TABLE IF NOT EXISTS pet_unlock (
    owner_uuid VARCHAR(36) NOT NULL,
    pet_id VARCHAR(64) NOT NULL,
    unlocked_at BIGINT NOT NULL,
    PRIMARY KEY (owner_uuid, pet_id)
)
CREATE TABLE IF NOT EXISTS pet_selection (
    owner_uuid VARCHAR(36) PRIMARY KEY,
    pet_id VARCHAR(64) NOT NULL
)
```

## サービス/マネージャー

- **`PetConfigRepository`** — `load(YamlConfiguration)`, `findById`, `getAll()`（`pets.yml`をインメモリのテンプレートへパース）
- **`PetOwnershipRepository`**（`SchemaOwner`） — `loadUnlocks()`（owner→解放済みpetIdの集合）, `loadSelections()`（owner→現在選択中のpetId）, `saveUnlock`, `saveSelection`
- **`PetManager`** — 現在召喚中のエンティティ（`LivingEntity`）をowner単位でインメモリ管理。`register`（既存があれば先にdespawn）, `despawn`, `hasActivePet`, `tickFollow()`（`FOLLOW_DISTANCE`＝4.0を超えて離れた/別ワールドの場合はオーナーへテレポートし、`Mob`ならターゲットを解除してMobが襲わないようにする。`PetModule`が10tickごとに呼び出す）, `despawnAll()`（`onDisable`から呼ばれる）
- **`PetService`** — `unlockedByOwner`/`selectedByOwner`をメモリキャッシュ（`loadAll()`で起動時ロード）。`ActionResult`（`OK`, `PET_NOT_FOUND`, `ALREADY_UNLOCKED`, `NOT_UNLOCKED`, `INSUFFICIENT_FUNDS`, `NO_ACTIVE_PET`）を返す：`unlock`（`economy.withdrawPlayer`）, `summon`（エンティティをスポーンしカスタム名表示、`getAllPets`/`getUnlockedPets`

## コマンド

`PetCommand` → `/ol pet [list|buy <id>|summon [id]|dismiss]`。`summon`に引数が無ければ最後に選択したペットを再召喚する。

## リスナー

`PetQuitListener`（`PlayerQuitEvent`） — 切断時に召喚中のペットを即座にdespawn。

## 消費する orelia-core / orelia-world API

`DatabaseManager`・Vaultの`Economy`（いずれも`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）。`rpg.api`は使用しない。
