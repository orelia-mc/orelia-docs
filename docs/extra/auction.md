# Auction モジュール（`rpg.extra.auction`）

DB永続化されたプレイヤー主導のオークションハウス。出品は期限付きで、期限切れは定期タスクが検出します。決済はVaultの`Economy`を直接使用します。

## ドメインモデル

```java
class AuctionListing {
    enum Status { ACTIVE, EXPIRED, COLLECTED }

    UUID id, sellerId;
    String sellerName;
    ItemStack item;
    double price;
    long listedAtMillis, expiresAtMillis;
    Status status;
    UUID buyerId;

    boolean isExpiredByTime(); // System.currentTimeMillis() >= expiresAtMillis
}
```

`Status` の意味：`ACTIVE`＝出品中、`EXPIRED`＝期限切れで未売却（出品者が回収待ち）、`COLLECTED`＝購入済みまたは出品者が回収済み。

## `auction_listing` テーブル

```sql
CREATE TABLE IF NOT EXISTS auction_listing (
    id VARCHAR(36) PRIMARY KEY,
    seller_uuid VARCHAR(36) NOT NULL,
    seller_name VARCHAR(32),
    item TEXT NOT NULL,
    price DOUBLE NOT NULL,
    listed_at BIGINT NOT NULL,
    expires_at BIGINT NOT NULL,
    status VARCHAR(16) NOT NULL,
    buyer_uuid VARCHAR(36)
)
```

`item` は `ItemStack` を `BukkitObjectOutputStream` でシリアライズしBase64化。`findAllActiveOrPending()` は `status != 'COLLECTED'` の行のみロードする。

## サービス/マネージャー

- **`AuctionRepository`**（`SchemaOwner`） — `findAllActiveOrPending()`, `save(AuctionListing)`（`status`/`buyer_uuid`をupsert）
- **`AuctionService`** — 全出品を `listingsById` にメモリキャッシュ（`loadAll()`で起動時ロード）。`ActionResult`（`OK`, `NOT_FOUND`, `ALREADY_RESOLVED`, `NOT_OWNER`, `CANNOT_BUY_OWN`, `INSUFFICIENT_FUNDS`, `INVALID_PRICE`, `EMPTY_HAND`, `INVENTORY_FULL`）を返す：
    - `list(seller, price, durationMillis)` — メインハンドのアイテムを出品（インベントリから即座に取り除く）
    - `buy(buyer, listingId)` — `economy.withdrawPlayer`/`depositPlayer` で即時決済し、アイテムをbuyerへ付与（入り切らなければドロップ）、`status`を`COLLECTED`に
    - `cancel(seller, listingId)` — `status`を`EXPIRED`にした上で`collect`を呼び出し、その場で回収させる
    - `collect(player, listingId)` — `EXPIRED`な自分の出品をインベントリへ回収（`INVENTORY_FULL`ガードあり）
    - `expireOverdueListings()` — `ACTIVE`かつ`isExpiredByTime()`な出品を`EXPIRED`へ。`AuctionModule`が`SchedulerService.runTimer`で1200tick（60秒）ごとに呼び出す
    - `getActiveListings()` / `getCollectable(UUID)`

## コマンド

`AuctionCommand` → `/ol auction`（引数無し、または`list`）でGUIを開く、`/ol auction sell <price>` で出品、`/ol auction collect` で期限切れ出品を一括回収。出品期間は固定 `DEFAULT_DURATION_MILLIS`＝3日。

## GUI

`AuctionGuiScreen.build(Player viewer)` — orelia-coreの`Gui`/`GuiButton`/`ItemBuilder`を使い54スロットの「オークション」画面を構築。自分の出品はクリックでキャンセル、他人の出品はクリックで購入。出品が無ければバリアブロックで「出品がありません」を表示。

## 消費する orelia-core / orelia-world API

`DatabaseManager`・Vaultの`Economy`（いずれも`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）。`rpg.api`は使用しない。
