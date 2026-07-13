# Effect（演出）の動作確認

対象: `rpg.effect`（[Effect モジュール仕様](../core/effect.md)）。パーティクル+サウンドの定義済みバンドルを名前付きIDで再生するだけの軽量モジュールです。

!!! warning "現状、orelia-core内部からは呼ばれていません"
    このドキュメント作成時点のコードベースでは、`skill`/`boss`/`accessory` などのモジュールが `EffectPlaybackService` を直接呼び出す実装にはなっていません。配線は `rpg.api.EffectApi`/`EffectApiImpl` 経由で外部プラグイン（想定利用元: `orelia-world` の `CutSceneModule`）向けにのみ用意されています。**レベルアップ時に `level_up` エフェクトが自動で鳴る、ボスのフェーズ移行で `boss_phase_change` が鳴る、といった「いかにも起きそうな」連携は現時点では起きません。** 動作確認の際にこれを「未実装」ではなく「バグ」と誤認しないよう注意してください。

## 前提条件

`effects.yml` 標準定義：

| ID | パーティクル | サウンド |
|---|---|---|
| `level_up` | TOTEM_OF_UNDYING ×40 | ENTITY_PLAYER_LEVELUP |
| `boss_phase_change` | EXPLOSION ×15 | ENTITY_WITHER_SPAWN |
| `quest_complete` | HAPPY_VILLAGER ×20 | ENTITY_EXPERIENCE_ORB_PICKUP |
| `enhancement_success` | ENCHANT ×30 | BLOCK_ANVIL_USE |

## 1. `EffectApi` 経由での再生確認

`orelia-core` 単体にはエフェクトを直接叩く player コマンドが無いため、確認には `rpg.api.EffectApi` を呼び出す外部プラグイン（テスト用の小さなプラグイン、または `orelia-world` の `CutSceneModule` 経由）が必要です。

1. `EffectApi.playAt(Location, "level_up")` を呼び出し、指定座標にTOTEM_OF_UNDYINGパーティクルとレベルアップサウンドが再生されることを確認する。
2. `EffectApi.playOnEntity(Entity, "boss_phase_change")` を呼び出し、対象エンティティの位置にEXPLOSIONパーティクル+WITHER_SPAWNサウンドが再生されることを確認する。
3. 存在しないIDを渡した場合にエラーで落ちず、何も再生されずに正常終了することを確認する。

## 2. 不正なenum名のフォールバック確認

`effects.yml` の `particle`/`sound` に存在しない `org.bukkit.Particle`/`org.bukkit.Sound` の名前を設定して `/oladmin reload` した場合:

1. `IllegalArgumentException` は捕捉され無視される仕様のため、サーバーがクラッシュしたりログにSEVEREエラーが出て他機能に波及したりしないことを確認する。
2. 該当エフェクトの再生を試みても、単に何も起きない（パーティクル/サウンドが出ない）だけで、他のエフェクトIDの再生には影響しないことを確認する。

## 3. `orelia-world` 連携での確認（`orelia-world` 導入時）

`orelia-world` の `CutSceneModule` がカットシーン演出として `EffectApi` を利用している場合、対応するカットシーンを実行し、演出中に定義済みエフェクトが正しいタイミングで再生されることを確認します。詳細は `orelia-world` の該当ドキュメント（`docs/world/cutscene.md`）を参照してください。

## 異常系・エッジケース

- 同じ座標で複数のエフェクトIDを短時間に連続再生し、パーティクル/サウンドが正しく重畳し、片方が消えたりクラッシュしたりしないこと。
- `enhancement_success` を強化未成功時に誤って鳴らしていないか（`orelia-world` のブラックスミスNPC実装側の呼び出し箇所を確認）。
