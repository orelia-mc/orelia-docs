# orelia-core 管理コマンド

すべて `/oladmin <サブコマンド>` の形で実行します。`/oladmin` 全体が `orelia.admin` 権限(デフォルトOP)で保護されています。`/oladmin help [ページ]` でサブコマンド一覧を確認でき、`/oladmin help <サブコマンド名>` と指定するとそのサブコマンド1件だけの説明を表示します。

## `reload`

```
/oladmin reload
```

全モジュールの設定ファイルを再読み込みします(サーバー再起動不要)。実行すると、そのリロードで実際に変更・追加・削除された設定の項目が一覧表示されます。設定ファイルを手で編集したあと、意図通り反映されたかをその場で確認できます。

## `spawn`

```
/oladmin spawn <モンスターID> [目安レベル]
```

自分の足元に指定したモンスターを1体湧かせます。`monsters.yml` に定義されたIDを指定します。目安レベルを省略すると、テンプレートどおりのレベル無し個体が湧きます。

## `spawnboss`

```
/oladmin spawnboss <ボスID>
```

自分の足元に指定したボスを湧かせます。`bosses.yml` に定義されたIDを指定します。

## `spawnpoint`

モンスターの自動湧きポイントを管理します。

```
/oladmin spawnpoint add <モンスターID> [間隔秒(既定30)] [最大同時数(既定3)] <目安レベル>
/oladmin spawnpoint remove <スポーンポイントUUID>
/oladmin spawnpoint list
```

- `add` は実行者の足元に登録します。目安レベルは必須です(省略すると使い方が表示されます)。指定したレベルに応じて、湧くモンスターのHP・攻撃力・防御力がスケーリングされ、名札にもレベルが表示されます。
- `list` で表示されるUUIDはクリックすると `remove` コマンドが自動入力されます。

## `dungeonblock`

ダンジョンの開放トリガーブロックを設置・解除します(いずれも、実行者が見ているブロックが対象です)。

```
/oladmin dungeonblock set <ダンジョンID>
/oladmin dungeonblock remove
/oladmin dungeonblock list [ページ]
```

`set` で指定するダンジョンIDは `dungeons.yml` に定義済みのものである必要があります。プレイヤーがそのブロックを右クリックすると、そのダンジョンを「発見」します。

## `dungeonarena`

ダンジョンの物理的な入場地点(挑戦開始時にプレイヤーが実際に移動させられる座標)を、実行者が今いる場所を使って登録・移動・削除します。`dungeons.yml` を手で編集してサーバーを再起動する必要はありません。`dungeonblock`(開放トリガーブロックの設置)とは別物で、こちらは「開放後、実際に挑戦を開始したときにどこへ送られるか」を扱います。1つのダンジョンに複数の入場地点を登録でき、挑戦開始時はその中からランダムに選ばれます。

```
/oladmin dungeonarena add <ダンジョンID>
/oladmin dungeonarena set <ダンジョンID> <番号>
/oladmin dungeonarena remove <ダンジョンID> <番号>
/oladmin dungeonarena list <ダンジョンID>
```

- `add` — 実行者の現在地を、指定ダンジョンの新しい入場地点として追加します。
- `set` — 指定ダンジョンの`<番号>`番目(`list`で表示される番号、1始まり)の入場地点を、実行者の現在地で上書きします。既存の地点を少しだけ動かしたいときは、`remove`してから`add`し直すよりこちらが手軽です。
- `remove` — `<番号>`番目の入場地点を削除します。そのダンジョンに残っている最後の1つは削除できません(入場地点が0件になると、`dungeons.yml`の古い形式(単一座標)へのフォールバック扱いになってしまうため)。
- `list` — 指定ダンジョンの入場地点を番号付きで一覧表示します。

## `npc`

NPCの設置・移動・削除を行います。

```
/oladmin npc create <ID> <タイプ> [entityType]
/oladmin npc move <ID>
/oladmin npc remove <ID>
/oladmin npc list [ページ]
/oladmin npc spawnall
```

- `create`/`move` は実行者の現在地に配置します。タイプは `WEAPON_SHOP`・`ARMOR_SHOP`・`ACCESSORY_SHOP`・`QUEST_RECEPTIONIST`・`JOB_CHANGE` など `npc.yml` の `type:` に対応する値です。
- `spawnall` は `npc.yml` に設定済みで、まだ出現していない全NPCをまとめて設置します(再実行しても重複しません)。職業指南役NPCだけは対象外です(下記`spawnnpc`参照)。

## `spawnnpc`

```
/oladmin spawnnpc <NPC-ID>
```

`spawnall` の対象外になっている職業指南役NPCなどを、実行者の足元に個別に手動配置します。

## `houseplot`

住居プロットを、実行者が今いる場所を使って登録・移動・削除します。`housing.yml` を手で編集してサーバーを再起動する必要はありません。`orelia-debug`導入時に使える`/oladmin house`(所持状態の付与・確認、[テストプレイ支援](debug-tools.md)参照)とは別物で、こちらはプロットの座標そのものを管理するコマンドです。

```
/oladmin houseplot register <ID> <価格> [表示名...]
/oladmin houseplot move <ID>
/oladmin houseplot remove <ID>
/oladmin houseplot list [ページ]
```

- `register` — 実行者の現在地を、新しい住居プロットとして登録します。表示名を省略するとIDがそのまま使われます。
- `move` — 既存のプロットの座標を、実行者の現在地に上書きします。
- `remove` — プロットを削除します。既に誰かが所有しているプロットは削除できません。
- `list` — 登録済みの全プロットをID順に、価格・座標付きで一覧表示します。

## `item`

```
/oladmin item give <プレイヤー名> <武器ID> [個数]
/oladmin item levelup [個数]
```

- `give` は指定したプレイヤーへ武器を配布します。テスト・報酬配布用のコマンドです。
- `levelup` は実行者が手に持っている武器のレベルを上げます。通常はキャラクターレベルに応じた上限までしか上げられませんが、[デバッグモード](debug-tools.md#debugmode)が有効な間は上限を無視して上げられ、`[個数]` で複数レベル分をまとめて適用できます。

## `gathering`

```
/oladmin gathering resetregen confirm
```

採取ブロックの再生成待ちタスクを、全ワールドまとめて取り消します。プレイヤーが建てた建築物には影響しません。`confirm` を付けずに実行すると確認メッセージのみ表示されます。

## `chat`

```
/oladmin chat <メッセージ>
```

管理者チャットチャンネルへメッセージを送信します。
