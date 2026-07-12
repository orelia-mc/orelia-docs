# 公開API（`rpg.api`）

`ApiModule` は常に最後に有効化されるモジュールで、`DatabaseModule, StatusModule, JobModule, ItemModule, AccessoryModule, SkillModule, EffectModule, MonsterModule, BossModule, GuiModule` を（`ModuleManager.get(...).orElseThrow` で）要求し、各モジュールのサービスを狭いインターフェースでラップして `ServicesManager`（`ServicePriority.Normal`）に公開します。

これが **`orelia-world` / `orelia-extra` にとって唯一の統合面**です。内部の `*Impl` クラスは規約上パッケージプライベート相当として扱われ、外部プラグインはインターフェース経由でのみ取得してください。

```java
Bukkit.getServicesManager().getRegistration(StatusApi.class).getProvider();
```

`PlayerDataManager` と `DatabaseManager` もラップせずそのまま公開されており、`orelia-world` は自分の `PlayerDataComponentLoader` やSQLテーブルをこれらに直接登録できます。

## `OreliaApi`

`OreliaPlugin` とモジュールの遅延参照をバックエンドに持つ横断的な参照用API。

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `getPlayerLevel(UUID playerId)` | `Optional<Integer>` | `StatusModule` 経由でレベルを取得 |
| `getPlayerJob(UUID playerId)` | `Optional<String>` | `JobModule` 経由で現在の職業名を取得 |
| `getPlayerStats(UUID playerId)` | `Map<String, Double>` | 装備・バフ適用後の最終ステータス（`StatType` 名がキー：HP, SP, ATK, DEF, AGI, DEX, INT, CRT, CRT_DMG, SPD）。未ロード時は空 |
| `getHeldWeaponId(UUID playerId)` | `Optional<String>` | メインハンドの `ItemStack` を武器IDへ解決。オフラインなら空 |
| `getAllWeaponIds()` | `Set<String>` | 全武器テンプレートIDの一覧 |

## `StatusApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `getLevel(UUID playerId)` | `Optional<Integer>` | レベル取得 |
| `getFinalStats(UUID playerId)` | `Map<String, Double>` | 最終ステータス |
| `addExperience(UUID playerId, long amount)` | `void` | 経験値付与（レベルアップ処理込み） |
| `tryConsumeSp(UUID playerId, double amount)` | `boolean` | SPを消費できれば消費してtrue |
| `damage(UUID playerId, double amount)` | `void` | HPを減らす |
| `heal(UUID playerId, double amount)` | `void` | HPを回復する |
| `getLeaderboard(int limit)` | `List<LeaderboardEntry>` | レベル降順（同レベルは経験値で判定）。オフラインプレイヤーも含む |

## `JobApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `getCurrentJob(UUID playerId)` | `Optional<String>` | 現在の職業 |
| `canUseWeaponType(UUID playerId, String weaponType)` | `boolean` | その武器種を使用可能か |
| `changeJob(UUID playerId, String jobName)` | `boolean` | 職業変更。`jobs.yml` に該当定義が無ければ false |

## `ItemApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `createWeapon(String weaponId)` | `Optional<ItemStack>` | 武器テンプレートから `ItemStack` を生成 |
| `identifyWeapon(ItemStack stack)` | `Optional<String>` | アイテムから武器IDを識別 |
| `getAllWeaponIds()` | `Set<String>` | 全武器ID |
| `weaponMeetsRequirements(UUID playerId, String weaponId)` | `boolean` | 職業/レベル要件を満たすか |
| `getEnhancementLevel(ItemStack stack)` | `int` | 現在の強化値 |
| `enhanceWeapon(ItemStack stack)` | `int` | 強化値を1上げ、新しい値を返す |

## `AccessoryApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `createAccessory(String accessoryId)` | `Optional<ItemStack>` | 装飾品テンプレートから `ItemStack` を生成 |
| `getAllAccessoryIds()` | `Set<String>` | 全装飾品ID |

## `SkillApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `grantSkillPoints(UUID playerId, int amount)` | `void` | スキルポイント付与 |
| `getSkillLevel(UUID playerId, String skillId)` | `int` | 指定スキルの習得レベル |

## `EffectApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `playAt(Location location, String effectId)` | `void` | 座標に定義済みエフェクトを再生 |
| `playOnEntity(Entity entity, String effectId)` | `void` | エンティティ位置にエフェクトを再生 |

## `CombatApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `identifyMonster(LivingEntity entity)` | `Optional<String>` | Oreliaのモンスターとしてタグ付けされていれば `monsters.yml` のIDを返す |
| `identifyBoss(String monsterId)` | `Optional<String>` | そのモンスターIDを包むボス定義があれば `bosses.yml` のIDを返す |

## `GuiApi`

| メソッド | 戻り値 | 説明 |
|---|---|---|
| `openStatus(Player player)` | `void` | ステータスGUIを開く |
| `openEquipment(Player player)` | `void` | 装備GUIを開く |
| `openSkill(Player player)` | `void` | スキルGUIを開く |
| `openJobChange(Player player)` | `void` | 職業変更GUIを開く |
| `openWarehouse(Player player)` | `void` | 倉庫GUIを開く |
| `openShop(Player player, List<ShopEntry> stock)` | `void` | 指定の在庫でショップGUIを開く |

## レコード型

```java
record LeaderboardEntry(UUID uuid, String name, int level, long experience) {}
record ShopEntry(String kind, String id, double price) {} // kind: "WEAPON" | "ACCESSORY" | "VANILLA"
```
