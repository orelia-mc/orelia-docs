# Ranking モジュール（`rpg.extra.ranking`）

レベルリーダーボードGUI。このモジュール自身はデータを一切所有せず、orelia-coreの`StatusApi`から取得した順位情報をそのまま表示するだけの薄いレイヤーです。永続化層・config・リスナーはありません。

## ドメインモデル

固有のモデルクラスは無く、orelia-coreの`rpg.api.LeaderboardEntry`（`name`, `level`, `experience`）をそのまま使用します。

## GUI

`RankingGuiScreen.build()` — orelia-coreの`Gui`/`GuiButton`/`ItemBuilder`を使い27スロットの「レベルランキング」画面を構築。`statusApi.getLeaderboard(27)`で取得した上位27件を順位付きで並べ、上位3位だけ`GOLD_BLOCK`/`IRON_BLOCK`/`COPPER_BLOCK`アイコン、4位以降は`PLAYER_HEAD`。各ボタンのクリックハンドラは何もしない（閲覧専用）。

## コマンド

`RankingCommand` → `/ol ranking` でGUIを開くのみ。

## 消費する orelia-core / orelia-world API

`StatusApi`（`ServicesManager`経由、ハード依存 — 無ければ`IllegalStateException`）のみ。`getLeaderboard`以外のメソッドは呼び出さない。
