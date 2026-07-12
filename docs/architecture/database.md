# データベース層

`DatabaseModule`（orelia-core）が SQLite または MySQL のいずれかで動く `DatabaseManager` を1つ構築し、全モジュールの repository がこれを共有して JDBC `Connection` を得ます。`DatabaseManager` 自体はスキーマを一切持たず、各 repository が `SchemaOwner` として自分のテーブルを作成・マイグレーションします。

## `DatabaseType`

```java
enum DatabaseType { SQLITE, MYSQL }
```

`parse(String raw, DatabaseType fallback)` — 大文字小文字を無視した `valueOf`。null/不正値はフォールバックへ。

## 接続構成（`config.yml: database.*`）

`DatabaseModule.onEnable` は `database.type`（デフォルト SQLITE）を読み、対応する `ConnectionProvider` を構築します。

- **SQLite**: `database.sqlite.file`（デフォルト `orelia.db`）→ `SQLiteConnectionProvider(dataFolder, file)`
- **MySQL**: `database.mysql.{host,port,database,username,password,use-ssl}` → `MySQLConnectionProvider(host, port, database, username, password, useSsl)`

その後 `databaseManager = new DatabaseManager(type, provider)`、`playerAccountRepository = new PlayerAccountRepository(databaseManager)` を構築し、`createSchemaIfNotExists()` を呼びます（失敗はログのみ、致命的にはしない）。`onDisable()` で `databaseManager.shutdown()`。

## `ConnectionProvider`

```java
interface ConnectionProvider {
    Connection getConnection() throws SQLException;
    void close();
}
```

- **`SQLiteConnectionProvider`** — 単一の長命な `synchronized` コネクションを保持（SQLite はシングルライター）。初回/失効時に `jdbc:sqlite:<絶対パス>` を開き、`PRAGMA foreign_keys = ON` と `PRAGMA busy_timeout = 5000` を設定。
- **`MySQLConnectionProvider`** — `getConnection()` 呼び出しごとに**新しい** JDBC コネクションを開く（Connector/J の `autoReconnect=true` に依存）。URL は `jdbc:mysql://host:port/database?useSSL=<useSsl>&autoReconnect=true&characterEncoding=UTF-8`。`close()` は no-op（呼び出し側が各コネクションを閉じる）。

!!! note "本番運用での注意"
    MySQL 接続はコネクションプーリングされていません。高同時接続数が想定される場合は HikariCP 等の実プールへの差し替えをコード側で検討する必要があります。

## `Repository<K, V>`

```java
interface Repository<K, V> {
    Optional<V> findById(K id);
    void save(V value);
    void delete(K id);
}
```

Repository は `DatabaseManager` かconfigファイルのみに触れ、Bukkit イベント／ゲームロジックには関与しません。

## `SchemaOwner`

```java
interface SchemaOwner {
    void createSchemaIfNotExists() throws SQLException;
}
```

各 repository のモジュール `onEnable` 内、`DatabaseManager` 構築後に1回呼ばれます。

## `PlayerAccountRepository`

`SchemaOwner` を実装（`Repository` ではない）。全モジュール共通の基礎テーブルを所有します。

```sql
players (uuid VARCHAR(36) PRIMARY KEY, name VARCHAR(16), first_join BIGINT, last_join BIGINT)
```

モジュール固有データはそれぞれのテーブルが uuid で JOIN する形で持ちます。`upsert(UUID uuid, String name, long nowMillis)` は方言別SQL（SQLite: `ON CONFLICT...DO UPDATE`、MySQL: `ON DUPLICATE KEY UPDATE`、`databaseManager.getType()` でswitch）を使用し、`SQLException` は `IllegalStateException` にラップされます。

## 方言分岐の実例

world側の player-state repository（例: `PlayerQuestRepository`）も同じパターンで、SQLite/MySQL 分岐は代表的に以下のようになります。

```java
switch (databaseManager.getType()) {
    case SQLITE -> "... ON CONFLICT(uuid) DO NOTHING";
    case MYSQL  -> "... INSERT IGNORE ...";
}
```
