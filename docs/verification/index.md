# 動作確認ガイド 概要

`orelia-core`（および連携する `orelia-world`）をサーバーに導入したあと、各機能が仕様どおりに動いているかを実機（テストサーバー）で確認するための手順集です。

コード上の仕様はソースコードを直接読んで書かれた [orelia-core モジュール一覧](../core/index.md) を正としています。ここに載っている手順はそれを実際にゲーム内で再現・確認するためのものであり、期待値もそのドキュメントの内容に合わせています。

## 読む順番

1. [プラグイン導入直後の確認](post-install.md) — まず最初に。プラグインが正しく起動し、DBスキーマが作成され、基本コマンドが応答することを確認します。
2. [Job（職業）](job.md) — 職業選択・武器種制限・パッシブ補正の確認。
3. [Skill（スキル）](skill.md) — 技（武器スキル）のソケット・習得・発動・クールダウン/SP消費の確認。
4. 以降は機能別に必要なものだけ確認してください：
    - [Item（武器）](item.md)
    - [Status（レベル/ステータス）](status.md)
    - [Accessory（装飾品）](accessory.md)
    - [Monster（モンスター）](monster.md)
    - [Boss（ボス）](boss.md)
    - [Economy（経済）](economy.md)
    - [GUI（各種画面）](gui.md)
    - [Effect（演出）](effect.md)

## 前提

- Paper 1.21.x サーバー、Java 21。
- 確認はOP権限を持つテストアカウントで行うことを想定しています（`/oladmin` は `orelia.admin`、デフォルトop）。
- 特記が無い限り、`orelia-core` 単体（`orelia-world` 未導入）でも確認できる項目のみを扱います。職業変更など `orelia-world` のNPCが必要な項目はその旨を明記しています。
- config は初期値（`items.yml`/`jobs.yml`/`skills.yml`/`monsters.yml`/`bosses.yml`/`accessories.yml`/`effects.yml` 同梱のデフォルト）のままである前提で、実際のID・数値を手順に記載しています。config を変更済みの場合はその内容に読み替えてください。

## 表記ルール

- `> command` はゲーム内チャット欄で打つコマンドです。
- 「期待結果」に書かれていない挙動が起きた場合は、対応する [orelia-core モジュールドキュメント](../core/index.md) の仕様と比較し、バグかconfig起因かを切り分けてください。
