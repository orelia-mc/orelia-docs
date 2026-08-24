# orelia-debug テストプレイ支援

`orelia-debug` は `orelia-core` に依存する別プラグインで、テストプレイ・バランス調整のための管理者コマンドを追加します。すべて `/oladmin <サブコマンド>` の形で、`orelia-core`本体のコマンドと同じ `/oladmin` に統合されています。

多くのコマンドは `[player]` を省略すると実行者自身が対象になります。`<core|world|extra>` という引数は、`orelia-core`統合前の3プラグイン構成の名残りで、内部的な設定の分類名として残っています(現在はすべて同じプラグインの機能です)。

## `gui`

```
/oladmin gui <status|equipment|skill|job|shop|warehouse|crafting|auction|mail|ranking|house|pet|achievement|dungeon|quest> [player]
```

指定した画面を強制的に開かせます。GUIの見た目を素早く確認したいときに使います。

## `money`

```
/oladmin money <give|set|take> [player] <金額>
```

所持金を付与/設定/引き出しします。

## `config`

```
/oladmin config <core|world|extra> list
/oladmin config <core|world|extra> view <ファイル名> [パス]
/oladmin config <core|world|extra> get <ファイル名> <パス>
/oladmin config <core|world|extra> set <ファイル名> <パス> <値>
/oladmin config <core|world|extra> save <ファイル名>
```

- `list` — そのカテゴリの設定ファイル名一覧を表示します。
- `view` — 指定ファイルをYAMLの木構造のまま、人間が読みやすい形式で表示します(パスを省略するとトップレベル全体、指定するとその配下だけ)。末端の値はクリックすると `set` コマンドが自動でチャット欄に入力されます。
- `get`/`set` — 特定のキー1つの値を確認/変更します。
- `save` — メモリ上の変更をファイルへ書き出します(`set`だけではファイルに保存されないため必要です)。

例: `/oladmin config core view monsters.yml forest_slime`

## `confighelp`

```
/oladmin confighelp <core|world|extra> <ファイル名>
```

指定した設定ファイルの全キーをフラットな一覧で表示します(`config view`より簡易な一覧表示)。

## `quest`

```
/oladmin quest complete [player] <クエストID>
/oladmin quest start [player] <クエストID>
/oladmin quest resetcooldown [player] <クエストID>
/oladmin quest list [player]
/oladmin quest ids
```

クエストの強制受注・強制達成・クールダウンリセットを行います。テストプレイで特定のクエスト状態を素早く再現したいときに使います。

## `title`

```
/oladmin title list [player]
/oladmin title grant [player] <称号>
/oladmin title equip [player] <称号>
/oladmin title unequip [player]
```

称号の確認・付与・装備・解除を行います。

## `dungeon`

```
/oladmin dungeon unlock [player] <ダンジョンID>
/oladmin dungeon forcestart [player] <ダンジョンID>
/oladmin dungeon forceend [player]
/oladmin dungeon status [player]
/oladmin dungeon ids
```

ダンジョンの開放・強制開始・強制終了・現在の状態確認を行います。

## `debugmode`

```
/oladmin debugmode <on|off|toggle> [player]
```

デバッグモードを切り替えます。有効な間は、武器の職業/レベル要件、スキルの武器種一致・ソケット・習得済み・クールダウン・SP消費、釣りざおの職業要件、武器レベルの上限([`/oladmin item levelup`](core-commands.md#item))が無視できます。成長系の上限(習得ポイントなど)そのものは変わりません。インメモリのみの設定で、再ログインするとリセットされます。

## `pet` / `mount` / `house`

```
/oladmin pet unlock [player] <ペットID>
/oladmin pet list [player]
/oladmin pet ids

/oladmin mount unlock [player] <乗り物ID>
/oladmin mount list [player]
/oladmin mount ids

/oladmin house grant [player] <土地ID>
/oladmin house clear [player]
/oladmin house status [player]
/oladmin house ids
```

いずれもお金のチェック無しで所持状態を付与・確認できます。

## `trade`

```
/oladmin trade status [player]
/oladmin trade forcecancel [player]
```

進行中の取引の状態確認・強制キャンセルを行います。

## `exp` / `skillpoints`

```
/oladmin exp give [player] <経験値量>
/oladmin skillpoints <give|set|take> [player] <量>
```

経験値・スキル習得ポイントを付与/設定/引き出しします。

## `relic`

```
/oladmin relic give [player] <ダンジョンID>
```

指定したダンジョン産のレリックを1個、経済チェック無しで付与します。

## `manual`

```
/oladmin manual [ページ]
```

`orelia-debug`が追加したコマンドの一覧を表示します。
