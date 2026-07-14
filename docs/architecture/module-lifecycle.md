# モジュールライフサイクル

## orelia-core

`rpg.core.OreliaPlugin`（`extends JavaPlugin`）が唯一のエントリーポイントです（`plugin.yml` の `main: rpg.core.OreliaPlugin`）。

### `onEnable()` の順序

1. `instance = this`（静的シングルトン、`getInstance()`）
2. `configManager = new ConfigManager(this)` → `configManager.register("config.yml")`
3. `schedulerService = new SchedulerService(this)`
4. `playerDataManager = new PlayerDataManager(getLogger(), schedulerService)`
5. `moduleManager = new ModuleManager(this)`
6. `PlayerConnectionListener(playerDataManager)` を Bukkit リスナーとして登録
7. `playerCommandRegistry` / `adminCommandRegistry`（いずれも `OlCommandRegistry` のサブクラス）を構築し、`ServicesManager.register(..., ServicePriority.Normal)` で公開
8. Bukkit コマンドをバインド：`/ol` → `OlRootCommand(playerCommandRegistry)`、`/oladmin` → `AdminCommand(this, adminCommandRegistry)`
9. **モジュール登録順（＝依存順）**：

    ```
    Database → Status → Job → Item → Skill → Accessory → Effect → Economy → Monster → Boss → Gui → Api（常に最後）
    ```

10. `moduleManager.enableAll()` — 登録順に各モジュールの `onEnable(plugin)` を呼び出す。モジュール単位で例外を捕捉・`SEVERE` ログ出力し、他モジュールの有効化は継続する。

### `onDisable()`

`playerDataManager.saveAllOnlineSync()`（非nullなら）→ `moduleManager.disableAll()`（登録順の**逆順**、モジュール単位で例外捕捉）→ `instance = null`。

### `reload()`（`/oladmin reload` から呼ばれる）

`configManager.reloadAll()` → `moduleManager.reloadAll()`（登録順、各モジュールの `onReload()` を呼ぶ。デフォルトは no-op）。

### モジュール間参照

`ModuleManager.get(Class<T>)` は `Map<Class<? extends RpgModule>, RpgModule>` から `Optional<T>` を返します。規約として、依存先モジュールは自分の `onEnable` 内で一度だけ取得し、存在しなければ `orElseThrow` で即座に `IllegalStateException` を投げること（遅延解決しない）。**あるモジュールが後に登録される別モジュールを参照してはならない。**

```java
public interface RpgModule {
    String getName();
    void onEnable(OreliaPlugin plugin);
    void onDisable();
    default void onReload() {}
}
```

## orelia-world

`rpg.world.core.OreliaWorldPlugin.onEnable()` が Composition Root です。

1. `ServicesManager` から orelia-core の `PlayerDataManager` を取得 — 見つからなければプラグインを disable してハードフェイル
2. 自前の `ConfigManager` / `SchedulerService` を構築（orelia-core のものとは別インスタンス。汎用インフラの実装を再利用しているだけ）
3. `WorldAdminCommand` を `/rpgworldadmin` にバインド
4. モジュールを **依存順**に登録（登録順＝有効化順、`WorldModuleManager` が管理）：

    ```
    RegionModule → DialogueModule → StoryModule → EventModule → CutSceneModule → DungeonModule → QuestModule → NpcModule
    ```

5. `moduleManager.enableAll()`

`reload()` は `configManager.reloadAll()` → `moduleManager.reloadAll()`。`WorldModule` インターフェースは orelia-core の `RpgModule` と同形（`getName()`, `onEnable(OreliaWorldPlugin)`, `onDisable()`, `onReload()` デフォルト no-op）ですが、`OreliaWorldPlugin` にパラメータ化された別インターフェースであり、共有はされていません。

!!! note "NpcModule が最後に登録される理由"
    `NpcModule` は `QuestModule` が既に登録済みであることに依存しています（クエスト受付NPCが `QuestProgressService` を参照するため）。新しいモジュールを追加する際は、この依存順を崩さないよう登録位置に注意してください。

## orelia-extra

`rpg.extra.core.OreliaExtraPlugin.onEnable()` が Composition Root です。

1. `ServicesManager` から orelia-core の `PlayerDataManager` と `PlayerCommandRegistry` / `AdminCommandRegistry` を取得 — 見つからなければプラグインを disable してハードフェイル（orelia-core が先に有効化されていることが前提）
2. 自前の `ConfigManager`（`config.yml` のみ登録） / `SchedulerService` を構築（orelia-core・orelia-world のものとは別インスタンス）
3. `ExtraAdminCommand` を orelia-core の `AdminCommandRegistry` へ `"extrareload"` として登録（`/oladmin extrareload`。orelia-core自身の `reload` や orelia-world の `worldreload` と衝突しないよう別名にしている）
4. モジュールをおおむねアルファベット順に登録（`ExtraModuleManager` が管理、登録順=有効化順、逆順で無効化）：

    ```
    Party → Guild → Trade → Mail → Auction → Housing → Pet → Mount → Ranking → Achievement
    ```

    モジュール同士に依存関係が無いためアルファベット順で登録されていますが、Ranking と Achievement だけは他モジュール（または orelia-core/orelia-world）が所有する状態を読むだけの立場のため最後に回されています。
5. `moduleManager.enableAll()`

`ExtraModule` インターフェース（`getName()`, `onEnable(OreliaExtraPlugin)`, `onDisable()`, `onReload()` デフォルト no-op）は orelia-core の `RpgModule` / orelia-world の `WorldModule` と同形ですが、`OreliaExtraPlugin` にパラメータ化された別インターフェースであり、共有はされていません。`reload()` は `configManager.reloadAll()` → `moduleManager.reloadAll()`。

## 各パッケージの共通レイヤー構造

`item`, `skill`, `job`, `status`, `accessory`, `monster`, `boss`, `effect`, `economy`, `gui`（core）、`quest`, `npc`, `dialogue`, `story`, `dungeon`, `region`, `cutscene`, `event`（world）、`party`, `guild`, `trade`, `mail`, `auction`, `housing`, `pet`, `mount`, `ranking`, `achievement`（extra）は、いずれも同じ内部レイヤーに従います。

- `repository/` — 純粋なデータアクセス。config駆動（`*.yml` をテンプレートへパース）または DB駆動（`Repository<K,V>` / `SchemaOwner` 実装）。Bukkit イベントやゲームロジックには触れない。orelia-extra では永続化を持たない Party のように、この層自体が無いモジュールもある。
- `model/` — 単純なデータ保持クラス（テンプレート、プレイヤーごとのコンポーネント）。
- `service/` または `manager/` — repository の上に乗るビジネスロジック。orelia-extra では永続化の無いモジュール（Party など）は `manager/` だけで状態を直接持つ。
- `listener/` — `onEnable` 内で配線される Bukkit イベントハンドラ。
- `command/` — 共有ディスパッチャ（`/ol`・`/oladmin` または `/rpgworldadmin`・`/rpgquest`・`/dialoguechoice`）に登録される `CommandExecutor`。独自の Bukkit トップレベルコマンドは持たない。
- `gui/` — GUI画面を持つモジュールのみ（core の Gui、world の Quest、extra の Auction/Mail/Ranking など）。

新しいモジュールを追加する場合は、この形（`QuestModule`/`DungeonModule`/`GuildModule` など）を踏襲し、`onEnable` 内で必要な依存を `ServicesManager`/`ModuleManager` から取得し、ハード依存が欠けていれば `IllegalStateException` で即座に失敗させます。
