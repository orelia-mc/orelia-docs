# Effect モジュール（`rpg.effect`）

軽量でconfig駆動のパーティクル+サウンドバンドル再生。他モジュールがハードコードされた `Particle`/`Sound` 呼び出しの代わりに、名前付きの合図（例: `level_up`）を発火できるようにします。

## ドメインモデル

```java
class EffectData {
    String id, particle;
    int particleCount;
    double spreadX, spreadY, spreadZ;
    String sound; // nullable
    float soundVolume, soundPitch;
}
```

タイミング/形状/アニメーションのフィールドは無く、単発の瞬間バーストのみです。

## `effects.yml` の実際のキー

```yaml
effects:
  level_up:
    particle: TOTEM_OF_UNDYING
    particle-count: 40
    spread-x: 0.6
    spread-y: 1.0
    spread-z: 0.6
    sound: ENTITY_PLAYER_LEVELUP
    sound-volume: 1.0
    sound-pitch: 1.0
```

その他: `boss_phase_change`, `quest_complete`, `enhancement_success`。デフォルト: particle `CLOUD`、count 15、spreads 0.5。

## サービス

- **`EffectRepository`** — `load`, `findById`, `getAll`
- **`EffectPlaybackService`**
    - `playAt(Location, effectId)`
    - `playOnEntity(Entity, effectId)`

    `world.spawnParticle(...)` の後、サウンドが設定されていれば `world.playSound(...)`。不正なenum名からの `IllegalArgumentException` は捕捉され無視されます。

## 他モジュールへの依存/被依存

現時点では `rpg.api.EffectApi`/`EffectApiImpl` 経由でのみ外部プラグイン向けに配線されています（想定される利用元: `orelia-world` の `CutSceneModule`）。この時点のコードベースでは、`skill`/`boss`/`accessory` などのモジュールが `EffectPlaybackService` を直接呼び出してはおらず、内部での採用待ちの独立機能です。
