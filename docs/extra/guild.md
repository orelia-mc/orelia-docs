# Guild モジュール（`rpg.extra.guild`）

DB永続化されたプレイヤー組織。leader/officer/memberの3ロールを持ち、officer以上が招待/追放、leaderのみが昇格/降格/解散を行えます。

## ドメインモデル

```java
class Guild {
    UUID id;
    String name, tag;
    UUID leaderId;
    Map<UUID, GuildRole> members; // LinkedHashMap

    static Guild create(String name, String tag, UUID leaderId); // leaderをLEADERとして追加
    GuildRole roleOf(UUID playerId);
    void addMember(UUID playerId, GuildRole role);
    void removeMember(UUID playerId);
    void setRole(UUID playerId, GuildRole role);
    void setLeaderId(UUID leaderId); // 新リーダーをLEADERロールに設定
}

enum GuildRole { LEADER, OFFICER, MEMBER }
```

## `guild` / `guild_member` テーブル

```sql
CREATE TABLE IF NOT EXISTS guild (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(64) NOT NULL,
    tag VARCHAR(16) NOT NULL,
    leader_id VARCHAR(36) NOT NULL
)
CREATE TABLE IF NOT EXISTS guild_member (
    guild_id VARCHAR(36) NOT NULL,
    uuid VARCHAR(36) NOT NULL,
    role VARCHAR(16) NOT NULL,
    PRIMARY KEY (guild_id, uuid)
)
```

## サービス/マネージャー

- **`GuildRepository`**（`SchemaOwner`） — `loadAll()`（guild + guild_member を結合してメモリに復元）, `save(Guild)`（guildをupsertし、guild_memberを全削除して再INSERT）, `delete(UUID guildId)`
- **`GuildManager`** — 全ギルドをメモリにキャッシュし、変更のたびに `repository.save`/`delete` へ書き込む write-through方式。`create`, `getByPlayer`, `invite`/`consumeInvite`/`clearInvite`（招待先→ギルドID、1人1件）, `addMember`, `removeMember`（メンバーが0人になったら自動解散）, `disband`, `persist(Guild)`
- **`GuildService`** — `ActionResult`（`OK`, `ALREADY_IN_GUILD`, `NOT_IN_GUILD`, `INSUFFICIENT_ROLE`, `TARGET_ALREADY_IN_GUILD`, `NO_PENDING_INVITE`, `CANNOT_TARGET_SELF`, `CANNOT_TARGET_LEADER`）を返す：`create`, `invite`（officer以上）, `accept`（MEMBERとして加入）, `leave`, `kick`（officer以上、対象がleaderなら拒否）, `setRole`（leaderのみ、promote→OFFICER / demote→MEMBER）, `disband`（leaderのみ）, `getGuild`

## コマンド

`GuildCommand` → `/ol guild <create <name> <tag>|invite <player>|accept|leave|kick <player>|promote <player>|demote <player>|disband|info>`。`info` は所属ギルドのタグ・名前とメンバー一覧（ロール付き）を表示。

## リスナー

`GuildQuitListener`（`PlayerQuitEvent`） — 切断したプレイヤー宛の未承諾招待をクリア。

## 消費する orelia-core / orelia-world API

`DatabaseManager`（`ServicesManager` 経由、ハード依存 — 無ければ `IllegalStateException`）のみ。`rpg.api`/`rpg.world.api` の呼び出しは無し。
