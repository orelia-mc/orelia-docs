# Event モジュール（`rpg.world.event`）

季節/ワールド/期間限定のウィンドウを定義し、EXP/所持金の倍率を与え、開始時に一度だけアナウンスをブロードキャストします。

## ドメインモデル

```java
class GameEventData {
    String id, name;
    GameEventType type;
    boolean recurring;
    MonthDay recurringStart, recurringEnd; // 年を問わない
    LocalDateTime oneOffStart, oneOffEnd;
    double bonusExpMultiplier, bonusMoneyMultiplier;
    String announceMessage;

    // 年をまたぐウィンドウ（例: 12/20〜1/5）も処理する
    boolean isActiveAt(LocalDateTime now);
}

enum GameEventType { SEASONAL, WORLD, LIMITED }
```

## `events.yml` の実際のキー

```yaml
events:
  halloween:
    name: "ハロウィンイベント"
    type: SEASONAL
    recurring: true
    start: "--10-25"
    end: "--10-31"
    bonus-exp-multiplier: 1.2
    bonus-money-multiplier: 1.2
    announce-message: "&6&lハロウィンイベントが始まりました！"
  anniversary_2026:
    type: LIMITED
    recurring: false
    start: "2026-07-01T00:00:00"
    end: "2026-07-14T23:59:59"
```

## サービス

- **`EventRepository`** — `load`/`getAll`（個別イベントのパース失敗はそのエントリのみ静かに無視され、`parse` が null を返す）
- **`EventScheduleService`**
    - `getActiveEvents()`
    - `isActive(id)`
    - `getBonusExpMultiplier()` / `getBonusMoneyMultiplier()` — 現在アクティブな全イベントの積
    - `findById`
- **`EventModule`** — 5分ごと（初回20tick後、以後 `20L*60*5` tick周期）に `checkForNewlyActiveEvents()` を実行し、新たにアクティブになったイベントの `announceMessage` をブロードキャスト（`lastAnnouncedActive` セットで連呼を防止）

## 消費する orelia-core API

`rpg.api.*` は使用しません。純粋な `Bukkit.broadcast` のみです。

!!! warning "報酬倍率は未配線"
    倍率は他モジュールが適用できるよう公開されていますが、コード内のコメントによれば**まだ quest/dungeon の報酬処理には組み込まれていません**。
