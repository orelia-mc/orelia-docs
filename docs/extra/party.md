# Party モジュール（`rpg.extra.party`）

サーバー再起動をまたいで永続化されない、インメモリのみのプレイヤーグループ機能。招待→承諾のハンドシェイクと、リーダー限定の招待/追放/解散を提供します。

## ドメインモデル

```java
class Party {
    UUID id;
    UUID leaderId;
    Set<UUID> members; // LinkedHashSet、リーダー自身を含む
    int maxSize;

    boolean isFull();
    boolean addMember(UUID playerId); // 満員なら false
    void removeMember(UUID playerId);
    boolean isEmpty();
}
```

`maxSize` は `config.yml` の `party.max-size`（既定 6）から `PartyModule.onEnable` で読み込まれ、`PartyService` に渡されます。

## マネージャー

`PartyManager` — `Party` 本体、`playerToParty`（逆引き）、`pendingInvites`（招待先→パーティID、1人1件まで）の3つの `ConcurrentHashMap` を保持。`create`, `getByPlayer`, `invite`, `consumeInvite`, `clearInvite`, `joinParty`, `leaveParty`（リーダーが抜けた場合は次のメンバーへ委譲、空になったら解散）, `disband`。

`PartyService` は `PartyManager` の上に権限チェック等のビジネスルールを実装し、`ActionResult` 列挙（`OK`, `ALREADY_IN_PARTY`, `NOT_IN_PARTY`, `NOT_LEADER`, `PARTY_FULL`, `TARGET_ALREADY_IN_PARTY`, `NO_PENDING_INVITE`, `CANNOT_TARGET_SELF`）を返します：`create`, `invite`（リーダーのみ）, `accept`, `leave`, `kick`（リーダーのみ）, `disband`（リーダーのみ）, `isSameParty(UUID, UUID)`, `getParty`。

永続化層（`repository/`）は無く、この機能だけで完結しています。

## コマンド

`PartyCommand` → `/ol party <create|invite|accept|leave|kick|disband|list>`。`list` はパーティメンバーをリーダーマーク付きで表示。

## リスナー

`PartyQuitListener`（`PlayerQuitEvent`） — 切断したプレイヤーの未承諾招待だけをクリアします。パーティ所属自体は明示的に `leave` するまで維持されます。

## 消費する orelia-core / orelia-world API

なし。`PartyModule.onEnable` は `plugin.getConfigManager()` と `plugin.getPlayerCommandRegistry()` のみを使用します。
