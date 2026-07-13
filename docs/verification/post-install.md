# プラグイン導入直後の確認

`orelia-core-1.0.0.jar`（と、使う場合は `orelia-world`）をサーバーの `plugins/` に置いて起動した直後に、最低限確認すべき項目です。上から順に確認してください。

## 1. 起動ログの確認

サーバーコンソール（または `logs/latest.log`）で以下を確認します。

- `OreliaCore` の有効化ログにエラー（`SEVERE`）が出ていないこと。
    - `ModuleManager.enableAll()` はモジュール単位で例外を捕捉するため、1モジュールが失敗しても他は動き続けます。**エラーが出ていないか必ずログを見る**（画面上プラグインは「有効」に見えても一部機能が死んでいる場合があるため）。
    - 特に `DatabaseModule` はスキーマ作成失敗をログのみで握りつぶすので注意（[データベース層](../architecture/database.md)参照）。
- `orelia-world` を併用する場合は `OreliaCore` より後に有効化されること（`depend: [OreliaCore]` で保証されるはず）。

!!! tip "モジュールが1つでも死ぬとどうなるか"
    例えば `Job` モジュールの有効化に失敗すると、後続の `Item`（職業要件チェックに依存）以降がおかしくなる可能性があります。ログにエラーが出ていたら、そのモジュールに対応する `docs/core/*.md` を見て依存関係を確認してください。

## 2. データフォルダとconfigの生成確認

`plugins/OreliaCore/` 配下に以下が生成されていることを確認します（`ConfigManager.register` がjar同梱のデフォルトをコピーします）。

```
plugins/OreliaCore/
  config.yml
  items.yml
  skills.yml
  jobs.yml
  accessories.yml
  monsters.yml
  bosses.yml
  effects.yml
  gui.yml
  messages.yml
  orelia.db          # database.type: SQLITE（デフォルト）の場合
```

- `database.type: MYSQL` にしている場合は `orelia.db` の代わりにMySQL側にテーブルが作られます。`config.yml` の `database.mysql.*` が正しいか、MySQL側の接続を確認してください（[データベース層](../architecture/database.md)）。
- `messages.yml` は生成されますが現状どのコードからも参照されません。空でも問題ありません。

## 3. コマンドの疎通確認

テストサーバーにOPで参加し、コマンドが応答することを確認します。

| コマンド | 期待結果 |
|---|---|
| `/ol` | 引数無しでサブコマンド一覧が表示される（少なくとも `item`, `job`, `status`） |
| `/ol job` | 「まだ選択されていません」的なメッセージ、またはエラーなく応答する |
| `/ol status` | ステータスGUI（27スロット、タイトル「ステータス」）が開く |
| `/oladmin` | 使用方法が表示される（OPでない場合は権限エラーメッセージが出ること） |
| `/oladmin reload` | エラー無く完了し、コンソールに例外が出ないこと |

`/oladmin reload` は全モジュールの config を再読込します（[Configシステム](../architecture/config.md)）。導入直後に一度実行し、エラーが出ないことを確認しておくと、config編集時の事故を早期発見できます。

## 4. プレイヤーデータの作成確認

新規プレイヤー（またはテストアカウント）でサーバーに参加し、退出します。

- DBの `players` テーブルに1行追加されていること（SQLiteなら `orelia.db` を `sqlite3` 等で開いて確認、または `/ol status` がエラー無く開けることで間接確認）。
- 再参加時、`/ol status` に表示される レベル/経験値/HP/SP が退出前と一致していること（永続化の確認）。

## 5. 最初の武器を入手して殴ってみる

1. `/ol job list` — `SWORDSMAN, SPEARMAN, WARRIOR, ARCHER` が表示される。
2. `orelia-world` のNPC（`JOB_CHANGE`）経由、または未導入の場合はDB/コードでの一時的な検証手段で `SWORDSMAN` になる（`orelia-core` 単体には職業変更コマンドが無いため。詳細は[Job の確認手順](job.md)）。
3. `/ol item give <自分の名前> wooden_training_sword 1` — 「見習いの剣」がインベントリに入る。
4. モブを殴り、ダメージが通ること（[Status の確認手順](status.md) のダメージ式に沿って計算されているか）。

ここまで確認できれば、Database → Status → Job → Item の基盤モジュールは一通り機能しています。続けて [Job](job.md) / [Skill](skill.md) など個別機能の確認に進んでください。

## よくある詰まりどころ

- **`/ol item give` が無反応 → コンソールにエラー**: `items.yml` のIDタイプミス。`wooden_training_sword` など実在するIDか確認（[Item の確認手順](item.md)）。
- **`/ol status` を開くとBARRIERしか出ない場面がある**: `SkillGuiScreen` は「識別可能な武器を手に持っていない」場合にBARRIERを表示する仕様（`StatusGuiScreen` ではなく `SkillGuiScreen` の話なので混同注意）。
- **経済プラグインとの連携がおかしい**: Vaultが導入されていないと `OreliaVaultEconomy` はそもそも登録されません（`EconomyModule.onEnable` はVault未導入時は静かにスキップ）。Vault連携を試す場合はVaultを先に導入してください。
