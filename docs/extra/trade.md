# Trade モジュール（`rpg.extra.trade`）

2人間のアイテム取引。共有GUIではなくコマンド駆動で、双方が `/ol trade confirm` するまで何も移動しません。サーバー再起動をまたいで永続化されない、インメモリのみの機能です（再起動で取引中セッションは消滅しますが、アイテムは提示した時点で既にプレイヤーのインベントリから引き抜かれているため、orelia-coreの通常のプレイヤーデータ保存で失われません）。

## ドメインモデル

```java
class TradeSession {
    UUID id;
    UUID playerA, playerB;
    TradeOffer offerA, offerB;

    UUID getOtherPlayer(UUID playerId);
    TradeOffer offerOf(UUID playerId);
    boolean involves(UUID playerId);
    boolean bothConfirmed();
}

class TradeOffer {
    List<ItemStack> items;
    boolean confirmed;

    void addItem(ItemStack item);   // 追加のたびに confirmed = false
    ItemStack removeItem(int index); // 削除のたびに confirmed = false
}
```

## マネージャー/サービス

- **`TradeManager`** — `sessionsByPlayer`（両プレイヤーが同じセッションを指す）と `pendingRequests`（対象→申込者）の2つの `ConcurrentHashMap`。`requestTrade`, `consumeRequest`, `clearRequest`, `start`, `getByPlayer`, `end`
- **`TradeService`** — `ActionResult`（`OK`, `ALREADY_TRADING`, `NOT_TRADING`, `NO_PENDING_REQUEST`, `CANNOT_TARGET_SELF`, `EMPTY_HAND`, `INVALID_SLOT`）を返す：
    - `request` / `accept` — 取引申し込みと開始
    - `addHeldItem` — メインハンドのアイテムをその場で提示（インベントリから即座に取り除く）
    - `removeOfferedItem(player, index)` — 提示を取り下げ、プレイヤーへ返却（`giveOrDrop`：入らなければ足元にドロップ）
    - `confirm(player)` — 片側を確定。両者確定済みなら `execute` を呼び、双方の提示アイテムを交換して `true` を返す
    - `cancel` / `forceCancelIfTrading`（切断時） — 双方の提示アイテムを持ち主へ返却してセッション終了

永続化層（`repository/`）は無く、この機能だけで完結しています。

## コマンド

`TradeCommand` → `/ol trade <player>|accept|add|remove <index>|confirm|cancel|view`。引数が既知のサブコマンドでなければオンラインプレイヤー名として扱い取引を申し込む。`view` は自分と相手の提示アイテムをスロット番号付きで表示。

## リスナー

`TradeQuitListener`（`PlayerQuitEvent`） — 切断側・相手側どちらの離脱でも即座に取引をキャンセルし、提示アイテムを返却。相手が受け取れなくても `forceCancelIfTrading` は必ず実行される。

## 消費する orelia-core / orelia-world API

なし。`TradeModule.onEnable` は `plugin.getPlayerCommandRegistry()` のみを使用します。
