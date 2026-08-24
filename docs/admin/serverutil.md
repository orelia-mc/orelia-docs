# orelia-serverutil

`orelia-serverutil` はRPG機能に依存しない、サーバー運用・UX向けの別プラグインです(ハブ転送・スコアボード/タブリスト表示・joinメッセージなど)。管理コマンドは `orelia-core`の`/oladmin`とは別の、独立した `/suadmin` を使います(権限: `orelia.serverutil.admin`、デフォルトOP)。

## `/suadmin reload`

```
/suadmin reload
```

設定ファイルを再読み込みします。

## `/suadmin setspawn`

```
/suadmin setspawn
```

実行者が今いる場所を、そのワールドのスポーン地点として設定します。

## `/suadmin worldsetup`

```
/suadmin worldsetup <ワールド名> [プロファイル名(既定: default)]
```

設定済みのワールドセットアッププロファイル(コマンド列)を、指定したワールドに対してまとめて実行します。ワールドがまだ存在しなくても指定できます(プロファイル自体がワールド作成コマンドを含んでいる場合など)。

## `/hub`

```
/hub
```

プレイヤー自身をハブ(サーバーまたはワールド、設定による)へ送ります。管理者専用ではなく、一般プレイヤーが使える案内コマンドです。
