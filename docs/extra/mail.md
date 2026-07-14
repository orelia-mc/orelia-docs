# Mail モジュール（`rpg.extra.mail`）

DB永続化されたメールボックス。アイテム添付に対応し、GUIで閲覧・受け取りを行います。`senderName` が null の場合は「System」からの送信として扱われ、将来的なオークションの「売却通知」など、他モジュールからのシステムメール送信を想定しています。プレイヤーが他プレイヤーへメールを送るコマンドは現状ありません（`MailService.send` はコード側からのみ呼び出せます）。

## ドメインモデル

```java
class MailMessage {
    UUID id, recipientId;
    String senderName; // null なら getSenderName() が "System" を返す
    String subject, body;
    ItemStack[] attachments;
    long sentAtMillis;
    boolean read, claimed;

    boolean hasAttachments();
}
```

## `mail_message` テーブル

```sql
CREATE TABLE IF NOT EXISTS mail_message (
    id VARCHAR(36) PRIMARY KEY,
    recipient_uuid VARCHAR(36) NOT NULL,
    sender_name VARCHAR(32),
    subject VARCHAR(64) NOT NULL,
    body VARCHAR(512),
    attachments TEXT,
    sent_at BIGINT NOT NULL,
    is_read BOOLEAN NOT NULL DEFAULT FALSE,
    claimed BOOLEAN NOT NULL DEFAULT FALSE
)
```

`attachments` は `ItemStack[]` を `BukkitObjectOutputStream` でシリアライズし Base64 化したもの（orelia-coreの倉庫機能と同じ手法）。

## サービス/マネージャー

- **`MailRepository`**（`SchemaOwner`） — `send(...)`（新規メッセージを作って`save`）, `findByRecipient(UUID)`（`sent_at DESC`）, `save(MailMessage)`（`is_read`/`claimed`/`attachments` を upsert）, `delete(UUID)`
- **`MailService`** — `send`, `getInbox(UUID)`, `markRead(MailMessage)`, `claim(Player, MailMessage)`（添付物をインベントリへ付与、入り切らない分はドロップ、既読かつ受取済みにする。既に受取済みなら `false`）, `delete`, `findById`

## コマンド

`MailCommand` → `/ol mail` でGUIを開く、`/ol mail unread` で未読件数のみ表示。

## GUI

`MailGuiScreen.build(Player)` — orelia-coreの `Gui`/`GuiButton`/`ItemBuilder` を使い54スロットの「メール」画面を構築。受信箱の各メールをアイコン（添付ありなら`CHEST`、無しなら`PAPER`）付きボタンとして並べ、クリックで既読化し、未受取の添付があれば受け取り処理を行う。

## 消費する orelia-core / orelia-world API

`DatabaseManager`（`ServicesManager` 経由、ハード依存 — 無ければ `IllegalStateException`）のみ。`rpg.gui.framework.GuiManager`/`Gui`/`GuiButton` は orelia-coreの汎用GUIフレームワークをそのまま利用（クリックはorelia-core側の`GuiListener`が処理）。
