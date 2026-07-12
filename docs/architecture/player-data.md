# プレイヤーデータ

`PlayerData`（`rpg.core.player`）は、UUIDをキーとした「オンラインプレイヤー1人分のクロスモジュール実行時状態」の入れ物です。Core自体はUUID/名前と `Map<Class<? extends PlayerDataComponent>, PlayerDataComponent>` しか管理せず、各コンポーネントの中身は関知しません。

## 型

```java
class PlayerData {
    UUID uuid;
    volatile String name;
    // ConcurrentHashMap<Class<? extends PlayerDataComponent>, PlayerDataComponent>

    <T extends PlayerDataComponent> void attach(Class<T> type, T component);
    <T extends PlayerDataComponent> Optional<T> component(Class<T> type);
    <T extends PlayerDataComponent> T require(Class<T> type); // 未アタッチなら IllegalStateException
}

interface PlayerDataComponent {
    UUID getOwner();
}

interface PlayerDataComponentLoader<T extends PlayerDataComponent> {
    Class<T> type();
    T loadOrCreate(UUID uuid); // join時、メインスレッド外で呼ばれる。nullを返してはならない
    void save(T component);    // quit時/自動保存時、メインスレッド外で呼ばれる
}
```

`require()` は、自モジュールのローダーが確実に実行済みであることを保証できる場合に使い、null化けによるサイレントなステータスバグを防ぐため「大声で」失敗します。

各モジュールは自分の `PlayerDataComponent`（例: `PlayerJobComponent`, `PlayerSkillComponent`, `PlayerStatusComponent`, `PlayerQuestComponent`, `PlayerDialogueComponent`, `PlayerStoryComponent`）と対応する `PlayerDataComponentLoader` を定義します。

## `PlayerDataManager`

`Map<UUID, PlayerData> online` と `List<PlayerDataComponentLoader<?>> loaders` を保持。

- `registerLoader(loader)` — 各モジュールが `onEnable` で自分のコンポーネント型のローダーを登録。
- `loadAsync(UUID uuid, String name, Runnable onLoaded)` — `SchedulerService` 経由で非同期実行。新しい `PlayerData` を構築し、登録済み全ローダーで `attachLoaded`（ローダー単位で例外捕捉・ログ）、`online` マップへ格納後、`onLoaded` をメインスレッドへ `scheduler.runSync` で戻す。
- `saveAndUnloadAsync(UUID uuid)` — 即座（同期）に `online` マップから削除し、その後 `saveAll` で全コンポーネントを非同期保存。
- `saveAllOnlineSync()` — オンライン全プレイヤーの `saveAll` を呼ぶ（プラグイン無効化時に使用するため、この経路だけは同期/ブロッキング）。
- `saveAll(PlayerData)` — 各ローダーについて、対応コンポーネントがあれば `loader.save(component)`（ローダー単位で例外捕捉・ログ）。
- `get(UUID) -> Optional<PlayerData>`

## Join/Quit ライフサイクル

`PlayerConnectionListener`（`OreliaPlugin.onEnable` で登録）が駆動します。

```java
@EventHandler(priority = EventPriority.LOWEST)
void onPreLogin(AsyncPlayerPreLoginEvent event) {
    playerDataManager.loadAsync(uuid, name, () -> {});
}

@EventHandler
void onQuit(PlayerQuitEvent event) {
    playerDataManager.saveAndUnloadAsync(uuid);
}
```

`AsyncPlayerPreLoginEvent` の時点でロードを開始するため、プレイヤーが完全にjoinする前にデータがキャッシュに載ります。

## orelia-world 側の統合

`orelia-world` はこの `PlayerDataManager` インスタンスを `ServicesManager` から取得し、自分の `PlayerQuestComponent`, `PlayerDialogueComponent`, `PlayerStoryComponent` などのローダーを同じマネージャーへ登録します。**世界側モジュールは自分でjoin/quitリスナーを持たず**、必ずこの仕組みに乗ります。
