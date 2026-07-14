# Status（レベル/ステータス）の動作確認

対象: `rpg.status`（[Status モジュール仕様](../core/status.md)）。他モジュールに依存しない基盤モジュールで、レベル/経験値、派生ステータス（HP, SP, ATK, DEF, CRT, CRT_DMG, SPD の7種。**AGI/DEX/INTは廃止済み**）、HP/SP自然回復を管理します。

!!! info "AGI/DEX/INTは削除されています"
    以前の `StatType` には AGI/DEX/INT も含まれていましたが、現在は削除され7種のみです。`/ol status` の画面や `config.yml: status.growth` にこれらが表示・設定されていても、それは古いconfigの残骸なので `/oladmin reload` 後も反映されません。表示されるのが7種で揃っていることをまず確認してください。

## 前提条件

`config.yml` の標準値（`exp-per-level: 100`, `max-level: 100`）を前提とします。

## 1. ステータスGUIの表示確認

```
/ol status
```

期待結果: 27スロットのGUI（タイトル「ステータス」）が開く。スロット4にプレイヤーヘッド（名前・レベル表示）、スロット10以降に各ステータス値が表示される。

## 2. レベルアップの確認

`StatusService.addExperience(UUID, long)` は while ループでレベルアップを処理します（`requiredExperience(level) = expPerLevel * level`、累積ではなく線形）。

1. 初期状態（レベル1, 経験値0）を `/ol status` で確認する。
2. モンスターを倒す、または（検証用に）経験値付与手段があれば付与し、レベル1→2に必要な経験値（`100 * 1 = 100`）を超えるまで蓄積させる。
3. レベルが上がった瞬間に:
    - `/ol status` の表示レベルが増える。
    - `LevelGrowthService.baseStatsForLevel` により全ステータスが再計算される（`base + perLevel * (level - 1)`。config.yml の `growth` セクション参照）。
    - 現在HP/SPが新しい最大値まで回復する（レベルアップ時の全回復仕様）。
    - `level_up` エフェクト（`TOTEM_OF_UNDYING` パーティクル + `ENTITY_PLAYER_LEVELUP` サウンド）が発火する場合は [Effect の確認手順](effect.md) と合わせて確認する。
4. 経験値を一気に大量付与し、1回のイベントで複数レベル分ジャンプする（例: レベル1→5）ケースでもループ処理により正しく最終レベルまで進むことを確認する。
5. `max-level`（デフォルト100）到達後は、それ以上経験値を積んでもレベルが上がらないことを確認する。

## 3. ダメージ計算式の確認

`CombatStatusListener` が `EntityDamageByEntityEvent` を処理します。

```
damage *= 1 + attackerATK / 100
reduction = victimDEF / (victimDEF + 100)
damage *= (1 - reduction)
```

1. ATK/DEFが分かっている状態（職業パッシブ・装備補正を含めた `/ol status` の最終値）で、モンスターへの1発分ダメージを実測し、上記式のとおりか電卓で検算する。
2. DEFが高い相手ほどダメージ軽減が大きくなる（式の反比例的な性質: DEF100で50%軽減）ことを、DEFの異なる2体のモンスターで比較確認する。

## 4. HP/SP自然回復の確認

`config.yml: status.regen`（デフォルト `hp-percent-per-tick: 0.5`, `sp-percent-per-tick: 1.0`, `period-ticks: 100` = 5秒間隔）。

1. HP/SPを意図的に減らす（モンスターに攻撃されるか、スキルを使ってSPを消費する）。
2. 何もせず約5秒待ち、`/ol status` を開き直してHPが最大値の0.5%、SPが1.0%回復していることを確認する（`currentHp += maxHp * pct / 100` でクランプされ、最大値を超えないこと）。

## 5. バフの適用順（フラット→パーセント）の確認

`StatModifier`（`FLAT`/`PERCENT`、`sourceKey`, `expiresAtMillis`）は装備コントリビューションとは別枠のバフです。バフを付与する外部トリガー（スキル・アクセサリ効果等、実装があれば）を使って:

```
finalValue = flatTotal * (1 + percentTotal / 100.0)
```

の順で計算されていることを確認します。0未満にはクランプされるため、大きな負のパーセントバフを与えてもステータスがマイナスにならないことも確認してください。バフは時間経過（`expiresAtMillis`）または `removeBuffsFromSource` で失効することも確認します（毎tickの `tickRegen` 呼び出し時に期限切れが剪定される）。

## 6. 永続化の確認

`StatusRepository` は `level, experience, current_hp, current_sp` のみをDBに保存します。装備コントリビューション/バフは実行時状態です。

1. レベルアップ・HP/SP変動後、一度サーバーから退出して再参加する。
2. `/ol status` のレベル/経験値/現在HP/SPが退出前と一致していること。
3. 装備コントリビューション（職業パッシブ、アクセサリ補正など）は再参加時に item/accessory/skill/job 各モジュールが再構築するため、装備状態が変わっていなければ再参加後も同じ最終値になっていることを確認する。

## 7. リーダーボードの確認

`StatusService.getLeaderboard()` を利用するUI/コマンドがあれば、レベル降順で正しく並んでいることを確認します（`orelia-core` に専用の表示コマンドが無い場合、この項目は `rpg.api.StatusApi` 経由で確認できる場合のみ対象とします）。
