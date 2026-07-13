# Economy（経済）の動作確認

対象: `rpg.economy`（[Economy モジュール仕様](../core/economy.md)）。プレイヤーごとに単一の個人残高を保持し、Vaultが導入されていれば Vault `Economy` プロバイダーとしても登録されます。

## 前提条件

`config.yml: economy.starting-balance`（デフォルト `100.0`）。

## 1. 初期残高の確認

1. 新規プレイヤー（DBに口座が無い状態）で最初に残高が参照される操作を行う（ショップNPCを開く、`/ol status` 等）。
2. 残高が `100.0`（starting-balance）で自動作成されることを確認する（`getBalance` は口座が無ければ starting balance で自動作成する仕様）。

## 2. 入金・出金の確認

`orelia-world` のNPC（ショップでの売却、モンスター撃破報酬など）を通して確認します。

1. モンスターを倒し、`money-min`〜`money-max` の範囲でお金が加算される（[Monster の確認手順](monster.md)）ことを残高の変化で確認する。
2. ショップNPC（`WEAPON_SHOP` 等）で武器を購入し、`price` 分だけ残高が減ることを確認する。
3. 残高が不足している状態で購入を試みると、`withdraw` が `false` を返して購入が失敗し、残高もアイテムも変化しないことを確認する。
4. 0以下の金額を入金しようとする操作（もしあれば）は no-op（残高が変化しない）であることを確認する。

## 3. 残高の直接設定・クランプの確認

`setBalance` はSQLite: `ON CONFLICT`、MySQL: `ON DUPLICATE KEY` のupsertで、0以上にクランプされます。管理者向けの残高操作コマンドが存在する場合は、負数を設定しようとして0にクランプされることを確認してください（`orelia-core` に標準の `/oladmin money` 的なコマンドは無いため、Vault経由の外部コマンド、または `rpg.api` 経由の操作がある場合のみ対象）。

## 4. Vault連携の確認

Vaultプラグインを導入した状態でサーバーを起動し、`OreliaVaultEconomy` が `ServicePriority.Normal` で登録されることを確認します。

1. Vault対応の他プラグイン（経済コマンド `/eco` 等を持つプラグイン、または `/balance` 系コマンド）から見た通貨名が `Gold`（`currencyNamePlural`/`currencyNameSingular`）になっていることを確認する。
2. `getName()` が `"Orelia"` として認識されること（複数の経済プラグインを併用していないか、Vaultのプロバイダー一覧などで確認）。
3. 小数点以下2桁（`fractionalDigits() = 2`）で残高が表示されることを確認する。
4. 銀行（bank）系コマンドを試し、`hasBankSupport() = false` により `NOT_IMPLEMENTED` として扱われる（エラーにはなるが、通常の個人残高には影響しない）ことを確認する。
5. オフラインプレイヤーの残高照会（`hasAccount` は常に `true`）が、プレイヤー名からのUUID解決（`Bukkit.getOfflinePlayer(name)`）を介して正しく行えることを確認する。

## 5. Vault未導入時の確認

Vaultを外した状態で再起動し、`EconomyModule.onEnable` がVault登録処理をスキップし、エラーなく有効化されることを確認します（`orelia-core` 単体の内部経済機能自体は問題なく動作するはずです）。

## 異常系・エッジケース

- 同一プレイヤーへの高頻度な入金/出金（連続撃破など）で残高が正しく累積し、取りこぼしや二重加算が起きないこと。
- `EconomyRepository` はキャッシュを持たず毎回DBへ直接読み書きするため、同時に複数の操作（購入と撃破報酬が同tickで発生するなど）が起きても最終残高が矛盾しないことを確認する。
