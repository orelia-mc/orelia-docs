# CutScene モジュール（`rpg.world.cutscene`）

カメラ移動/チャットメッセージ/タイトル/エフェクトの時限シーケンスをスクリプト化し、プレイヤーごとにスケジューラ経由で再生します。

## ドメインモデル

```java
class CutSceneData {
    String id;
    List<CutSceneStep> steps;
}

class CutSceneStep {
    CutSceneStepType type;
    long delayTicks;
    String message, title, subtitle, effectId;
    String cameraWorld; double cameraX, cameraY, cameraZ, cameraYaw, cameraPitch;
}

enum CutSceneStepType { CAMERA, MESSAGE, EFFECT, TITLE } // BGMは今後の追加予定としてコメントあり
```

## `cutscenes.yml` の実際のキー

```yaml
cutscenes:
  goblin_king_intro:
    steps:
      step_1: { type: TITLE, delay-ticks: 0, title: "&4ゴブリンキング", subtitle: "&7洞窟の奥に潜む脅威" }
      step_2: { type: MESSAGE, delay-ticks: 20, message: "&7地響きとともに、巨大な影が姿を現した..." }
      step_3: { type: EFFECT, delay-ticks: 40, effect-id: boss_phase_change }
```

## サービス

- **`CutSceneRepository`** — `load`/`findById`
- **`CutScenePlaybackService.play(Player, cutsceneId)`** — 各ステップを `SchedulerService.runLater(delayTicks)` でスケジュール。`runStep` が種別ごとに処理：
    - `MESSAGE` — チャット送信
    - `TITLE` — Adventure `Title`
    - `EFFECT` — `EffectApi.playOnEntity`（nullable）
    - `CAMERA` — 簡易的なテレポート+視線設定（本物の分離スペクテイターカメラではない旨、明示的にコメントあり）

## 消費する orelia-core API

`EffectApi`（nullable。取得はするがハード依存ではなく、他モジュールのような `IllegalStateException` ガードはこのモジュールには無い）。
