# コマンド一覧(管理者用)

## `/oladmin`(orelia-core、権限: `orelia.admin`)

| コマンド | 説明 |
|---|---|
| `reload` | 全設定を再読み込み(差分を表示) |
| `spawn <id> [lv]` | モンスターを湧かせる |
| `spawnboss <id>` | ボスを湧かせる |
| `spawnpoint add\|remove\|list` | 自動湧きポイント管理 |
| `dungeonblock set\|remove\|list` | ダンジョン開放ブロック管理 |
| `npc create\|move\|remove\|list\|spawnall` | NPC管理 |
| `spawnnpc <id>` | 職業指南役NPCなどを個別設置 |
| `item give\|levelup` | 武器配布・武器レベル操作 |
| `gathering resetregen confirm` | 採取再生成タスクの一括取消 |
| `chat <message>` | 管理者チャット送信 |

詳細は [orelia-core 管理コマンド](core-commands.md) を参照。

## `/oladmin`(orelia-debug、同じ入口に追加登録)

| コマンド | 説明 |
|---|---|
| `gui <画面> [player]` | GUIを強制表示 |
| `money give\|set\|take` | 所持金操作 |
| `config <core\|world\|extra> list\|view\|get\|set\|save` | 設定ファイルの確認・編集 |
| `confighelp <core\|world\|extra> <file>` | 設定ファイルの全キー一覧 |
| `quest complete\|start\|resetcooldown\|list\|ids` | クエスト操作 |
| `title list\|grant\|equip\|unequip` | 称号操作 |
| `dungeon unlock\|forcestart\|forceend\|status\|ids` | ダンジョン操作 |
| `debugmode on\|off\|toggle` | デバッグモード切り替え |
| `pet unlock\|list\|ids` | ペット付与 |
| `mount unlock\|list\|ids` | 乗り物付与 |
| `house grant\|clear\|status\|ids` | 住居付与 |
| `trade status\|forcecancel` | 取引状態確認・強制キャンセル |
| `exp give` | 経験値付与 |
| `skillpoints give\|set\|take` | スキル習得ポイント操作 |
| `relic give` | レリック付与 |
| `manual [page]` | orelia-debugのコマンド一覧 |

詳細は [orelia-debug テストプレイ支援](debug-tools.md) を参照。

## `/suadmin`・`/hub`(orelia-serverutil、権限: `orelia.serverutil.admin`)

| コマンド | 説明 |
|---|---|
| `/suadmin reload` | 設定再読み込み |
| `/suadmin setspawn` | ワールドスポーン地点を設定 |
| `/suadmin worldsetup <world> [profile]` | ワールドセットアッププロファイルを実行 |
| `/hub` | ハブへ転送(プレイヤーも使用可) |

詳細は [orelia-serverutil](serverutil.md) を参照。
