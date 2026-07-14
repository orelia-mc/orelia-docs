# プラグイン導入直後の確認

`orelia-core-1.0.0.jar`（と、使う場合は `orelia-world`）をサーバーの `plugins/` に置いて起動した直後に、最低限確認すべき項目です。上から順に確認してください。

## 1. 起動ログの確認

サーバーコンソール（または `logs/latest.log`）で以下を確認します。

- `OreliaCore` の有効化ログにエラー（`SEVERE`）が出ていないこと。
    - `ModuleManager.enableAll()` はモジュール単位で例外を捕捉するため、1モジュールが失敗しても他は動き続けます。**エラーが出ていないか必ずログを見る**（画面上プラグインは「有効」に見えても一部機能が死んでいる場合があるため）。
    - 特に `DatabaseModule` はスキーマ作成失敗をログのみで握りつぶすので注意（[データベース層](../architecture/database.md)参照）。
- `orelia-world` を併用する場合は `OreliaCore` より後に有効化されること（`depend: [OreliaCore]` で保証されるはず）。

!!! tip "モジュールが1つでも死ぬとどうなるか"
    例えば `Job` モジュールの有効化に失敗すると、後続の `Gathering`/`Item`（職業要件チェックに依存）以降がおかしくなる可能性があります。ログにエラーが出ていたら、そのモジュールに対応する `docs/core/*.md` を見て依存関係を確認してください。現在のモジュール登録順（＝有効化順）は `Database → Status → Job → Gathering → Item → Skill → Accessory → Effect → Economy → Monster → Boss → Gui → Api` です（`Gathering` がJobの直後、Itemの手前に追加されています）。

!!! note "WorldGuardはオプション（softdepend）"
    `orelia-core` の `plugin.yml` には `softdepend: [WorldGuard]` が追加されています。WorldGuardは必須ではありません。導入していない場合、コンソールに「WorldGuard not found; gathering/farming will not respect region protection.」という情報ログが出るのが正常です（採取/農業のリージョン保護判定が常に許可扱いになるだけで、他機能には影響しません）。

## 2. データフォルダとconfigの生成確認

`plugins/OreliaCore/` 配下に以下が生成されていることを確認します（`ConfigManager.register` がjar同梱のデフォルトをコピーします）。

```
plugins/OreliaCore/
  config.yml
  items.yml
  skills.yml
  jobs.yml
  accessories.yml
  gathering.yml
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
| `/ol` | 引数無しでサブコマンド一覧が表示される（少なくとも `item`, `job`, `status`, `gathering`） |
| `/ol job` | 「まだ選択されていません」的なメッセージ、またはエラーなく応答する |
| `/ol status` | ステータスGUI（27スロット、タイトル「ステータス」）が開く |
| `/ol gathering` | 採取/農業レベルと一括範囲半径が表示される（初期状態はレベル1・半径0） |
| `/oladmin` | 使用方法が表示される（OPでない場合は権限エラーメッセージが出ること） |
| `/oladmin reload` | エラー無く完了し、コンソールに例外が出ないこと |

`/ol`・`/oladmin` とも、途中まで打ってTabキーを押すとサブコマンド名が補完されることも合わせて確認しておくと良いです（`OlRootCommand`/`AdminCommand` が `TabCompleter` を実装済み）。

`/oladmin reload` は全モジュールの config を再読込します（[Configシステム](../architecture/config.md)）。導入直後に一度実行し、エラーが出ないことを確認しておくと、config編集時の事故を早期発見できます。

## 4. プレイヤーデータの作成確認

新規プレイヤー（またはテストアカウント）でサーバーに参加し、退出します。

- DBの `players` テーブルに1行追加されていること（SQLiteなら `orelia.db` を `sqlite3` 等で開いて確認、または `/ol status` がエラー無く開けることで間接確認）。
- 再参加時、`/ol status` に表示される レベル/経験値/HP/SP が退出前と一致していること（永続化の確認）。

## 5. 最初の武器を入手して殴ってみる

1. `/ol job list` — `FENCER, WARRIOR, ARCHER` の3種が表示される（`SPEARMAN` は廃止済み、`WARRIOR` はAXE専用になっています。詳細は[Job の確認手順](job.md)）。
2. 職業に就くには `orelia-world` の `JOB_CHANGE` NPC（`job_master`）が必要です。このNPCは起動時に自動配置されないため、まず `/oladmin spawnnpc job_master` で1体配置してから話しかけてください（`orelia-core` 単体には職業変更コマンドが無いため。詳細は[Job の確認手順](job.md)）。
3. `FENCER` になった状態で `/ol item give <自分の名前> wooden_training_sword 1` — 「見習いの剣」がインベントリに入る。
4. モブを殴り、ダメージが通ること（[Status の確認手順](status.md) のダメージ式に沿って計算されているか）。

ここまで確認できれば、Database → Status → Job → Item の基盤モジュールは一通り機能しています。続けて [Job](job.md) / [Skill](skill.md) / [Gathering](gathering.md) など個別機能の確認に進んでください。

## 6. `orelia-world` 併用時の追加確認

`orelia-world` を導入している場合、参加直後にホットバー右端へ「プレイヤー情報」ネザースターが自動的に固定配置されます。これを右クリックしてメニュー（クエスト/ジョブ/スキル/実績）が開くことを確認してください。特に「スキル」タブは、`orelia-core` 単体では到達手段の無い `SkillGuiScreen`（武器スキルのソケット/習得画面）への唯一の導線になっています（詳細は [Skill の確認手順](skill.md)）。

## よくある詰まりどころ

- **`/ol item give` が無反応 → コンソールにエラー**: `items.yml` のIDタイプミス。`wooden_training_sword` など実在するIDか確認（[Item の確認手順](item.md)）。
- **職業指南役NPCが見当たらない**: `JOB_CHANGE` タイプNPCだけは起動時自動配置の対象外です。`/oladmin spawnnpc job_master` で手動配置してください（他のNPCは自動配置されるので、この点だけ特別です）。
- **`SkillGuiScreen` が開けない**: `orelia-core` 単体には到達経路がありません。`orelia-world` の「プレイヤー情報」ネザースター（スキルタブ）経由で開いてください（[Skill の確認手順](skill.md)）。
- **経済プラグインとの連携がおかしい**: Vaultが導入されていないと `OreliaVaultEconomy` はそもそも登録されません（`EconomyModule.onEnable` はVault未導入時は静かにスキップ）。Vault連携を試す場合はVaultを先に導入してください。
- **採取/農業でブロックが一括破壊されない**: `/ol gathering` で現在のレベルと一括範囲半径を確認してください。半径0（Lv1-9）の間はシフト+破壊/収穫でも1マスしか処理されません（[Gathering の確認手順](gathering.md)）。
