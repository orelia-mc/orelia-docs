# orelia-core 概要

`orelia-core` は Paper 1.21.x (Java 21) 向けプラグインで、Orelia 3プラグイン構成の基盤です。

- **orelia-core**（本体）: Core, Item, Skill, Job, Status, Accessory, Monster, Boss, Effect, Economy, GUI, Gathering, Database, API, Util
- `orelia-world`: このプラグインに依存し、`rpg.api` 経由でのみ通信
- `orelia-extra`（未実装）: 同様に依存予定

## モジュール登録順（＝依存順）

```
Database → Status → Job → Gathering → Item → Skill → Accessory → Effect → Economy → Monster → Boss → Gui → Api（常に最後）
```

詳細は[モジュールライフサイクル](../architecture/module-lifecycle.md)を参照してください。

## モジュール一覧

| モジュール | 概要 |
|---|---|
| [公開API (rpg.api)](api.md) | 他プラグイン向けの唯一の統合面。`ServicesManager` 経由で公開 |
| [Item](item.md) | 武器テンプレート、武器生成・識別・強化 |
| [Skill](skill.md) | 武器スキル、実行アーキタイプ（executor）、ソケット/習得 |
| [Job](job.md) | 職業（剣士/槍士/戦士/弓士/釣り人）と武器種制限・パッシブ補正 |
| [Status](status.md) | レベル/経験値、ステータス計算式、HP/SP自然回復 |
| [Gathering](gathering.md) | 採掘・木こりのブロック復活、農業の一括植え付け/収穫、共有レベル/範囲拡張システム |
| [Accessory](accessory.md) | 装飾品スロット（護符/指輪/首飾り/翼）とステータス加算 |
| [Monster](monster.md) | モンスターテンプレート、スポーン、スポーンポイント、ドロップ報酬 |
| [Boss](boss.md) | フェーズ演出、狂暴化、周期アビリティ発動（Monsterの上位レイヤー） |
| [Effect](effect.md) | パーティクル+サウンドの定義済みバンドル再生 |
| [Economy](economy.md) | 個人残高管理、Vault `Economy` プロバイダー |
| [GUI](gui.md) | 共通GUIフレームワークとステータス/装備/スキル/職業/ショップ/倉庫画面 |

## Config ファイル

`items.yml`, `skills.yml`, `jobs.yml`, `accessories.yml`, `monsters.yml`, `bosses.yml`, `effects.yml`, `gui.yml`, `gathering.yml`, `config.yml`, `messages.yml`（未使用）— 詳細は[Configシステム](../architecture/config.md)。

## リロード

`/oladmin reload` で全モジュールの設定ファイルを再読込します。
