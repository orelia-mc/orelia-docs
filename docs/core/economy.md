# Economy モジュール（`rpg.economy`）

プレイヤーごとに単一の個人残高を保持するウォレット/通貨モジュール。Vaultが存在する場合、自身を Vault `Economy` プロバイダーとして登録し、サードパーティプラグインが Orelia のクラスに依存せず取引できるようにします。

## データ

`EconomyRepository` は呼び出しのたびに直接DBへ読み書きします（インメモリキャッシュ無し）。Vaultのインターフェースはオフラインプレイヤーにも応答する必要があるためです。

```sql
player_economy (uuid VARCHAR(36) PRIMARY KEY, balance DOUBLE NOT NULL DEFAULT 0)
```

## サービス

- **`EconomyRepository`**
    - `hasAccount(UUID)`
    - `createAccountIfMissing(UUID)`（`startingBalance` で初期化）
    - `getBalance(UUID)`（口座が無ければ starting balance で自動作成）
    - `setBalance(UUID, double)`（SQLite: `ON CONFLICT`、MySQL: `ON DUPLICATE KEY` のupsert。0以上にクランプ）
- **`EconomyService`**
    - `getBalance(UUID)`
    - `has(UUID, double)`
    - `deposit(UUID, double)`（0以下ならno-op）
    - `withdraw(UUID, double)` → `boolean`（残高不足ならfalse）
    - `setBalance(UUID, double)`
- **`OreliaVaultEconomy implements net.milkbowl.vault.economy.Economy`** — `EconomyService` に対する完全なVaultアダプター
    - `getName()` = `"Orelia"`
    - `currencyNamePlural()`/`currencyNameSingular()` = `"Gold"`
    - `fractionalDigits()` = `2`
    - `hasBankSupport()` = `false`（銀行系メソッドは全て `EconomyResponse.ResponseType.NOT_IMPLEMENTED`）
    - `hasAccount(...)` は常に `true`
    - プレイヤー名→UUID解決は `Bukkit.getOfflinePlayer(name).getUniqueId()`

## `EconomyModule.onEnable`

`config.yml: economy.starting-balance`（デフォルト 100.0）を読み込み、"Vault" プラグインが存在する場合のみ `OreliaVaultEconomy` を `Bukkit.getServicesManager()` に `ServicePriority.Normal` で登録します。

## 他モジュールへの依存/被依存

`DatabaseModule` を必須とします。`monster`（撃破時のお金報酬）、`gui`（ショップの購入/売却）、および外部プラグイン（Vault経由）から利用されます。
