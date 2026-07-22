# orelia-serverutil 概要

`orelia-serverutil` は Orelia RPGプラグイン群（orelia-core / orelia-world / orelia-extra）とは**独立した**、サーバー運用・UX周りの汎用機能プラグインです。ゲームプレイに依存しないため、Orelia系プラグインが一切導入されていないサーバーでも単体で動作します。

## なぜRPGスイートから切り離されているか

`orelia-world`/`orelia-extra`/`orelia-debug` は `orelia-core` を `depend`（必須依存）として扱い、`rpg.core.config.ConfigManager`/`rpg.core.message.MessageManager` をそのまま再利用します。一方 `orelia-serverutil` にとって OreliaCore は **softdepend（任意依存）** に過ぎません。OreliaCoreが未導入のサーバーで `rpg.core.*` を直接importすると起動時に `NoClassDefFoundError` を起こしかねないため、`rpg.serverutil.paper.config`/`rpg.serverutil.paper.message`/`rpg.serverutil.paper.util.ColorUtil` は独立した軽量な自前実装になっています。同じ理由で、`/ol`・`/oladmin`・`/rpgworldadmin` といった共有コマンドディスパッチャーにも登録されず、独自のトップレベルBukkitコマンド `/hub`・`/suadmin` を宣言します（後述）。

## モジュール構成（3つのGradleモジュール）

- **common**（`rpg.serverutil.common`） — Paper/Velocity APIに依存しない、プラットフォーム非依存のプロトコル/データクラス群。paper・velocity双方から共有されます。
- **paper**（`rpg.serverutil.paper`） — 本体となるPaper/Bukkitプラグイン。単体で完全に動作し、OreliaCoreはsoftdependのみ。
- **velocity**（`rpg.serverutil.velocity`） — Velocityプロキシ側プラグイン。`hub.mode: PROXY` とサーバー切り替え時のjoin/leaveメッセージ演出にのみ必要で、paper側は無くても機能します。

## Paper側モジュール一覧（登録順＝有効化順、逆順で無効化）

```
SpawnModule → WorldSetupModule → VelocityBridgeModule → HubModule →
ScoreboardModule → TabListModule → BelownameModule → JoinMessageModule →
AnnounceModule → ChatModule → HealthCheckModule → CoreIntegrationModule（常に最後）
```

| モジュール | 概要 |
|---|---|
| SpawnModule | `/suadmin setspawn` によるワールドスポーン地点設定（`World#setSpawnLocation`） |
| WorldSetupModule | `/suadmin worldsetup` で `world-setup.profiles.<profile>.gamerules` を対象ワールドへ一括適用 |
| VelocityBridgeModule | Velocityとのプラグインメッセージング橋渡し（`orelia:serverutil`チャンネル）。`HubModule`はPROXYモード時にこのモジュールを参照する |
| HubModule | `/hub` の実処理（`HubService`）。TELEPORT/PROXYモードの切り替え |
| ScoreboardModule | サイドバースコアボード。`ScoreboardApi`を公開 |
| TabListModule | タブリスト名装飾（prefix/suffix/color）と右側の値。`TabListApi`を公開 |
| BelownameModule | 名札下テキスト。`BelownameApi`を公開 |
| JoinMessageModule | 通常join/quitメッセージ、およびサーバー切り替え時のjoin/leaveメッセージ |
| AnnounceModule | join後の歓迎メッセージ・タイトル表示 |
| ChatModule | チャットフォーマット・プレースホルダー・ホバーツールチップ・カラーコード許可。`ChatApi`を公開 |
| HealthCheckModule | op向けのjoin時TPS/オンライン人数サマリー表示（および導入プラグインのバージョン表示） |
| CoreIntegrationModule | OreliaCore導入時、レベル/職業/所持金/HPを上記の各APIへ自動的にプロバイダー登録する。常に最後に有効化される |

`ServerUtilModuleManager` は orelia-core の `RpgModule`/`ModuleManager` と同様のライフサイクルパターン（登録＝有効化順、逆順で無効化）を踏襲しています。`register()` は全モジュールを先にタイプ別マップへ登録してから `enableAll()` を実行するため `get(X.class)` は登録順によらず解決できますが、モジュール自身のフィールドは自分の `onEnable()` が実行されるまで未初期化です。`CoreIntegrationModule` が最後に置かれているのは、`ScoreboardApi`/`TabListApi`/`BelownameApi`/`ChatApi` に加えOreliaCore側の `StatusApi`/`EconomyApi`/`JobApi` まで、それより前に有効化された全APIに依存するためです。

## コマンド

orelia-world/orelia-extra/orelia-debugと異なり、OreliaCoreの共有 `PlayerCommandRegistry`/`AdminCommandRegistry` には登録されません（OreliaCoreはsoftdependであり、レジストリ自体が存在しない可能性があるため）。`plugin.yml` で独自のトップレベルBukkitコマンドとして宣言されます。

### `/hub`

`hub.mode` の値で挙動が変わります。

- `TELEPORT`（デフォルト） — このサーバー内の `hub.teleport.*` 座標へテレポート。単体サーバーでもそのまま使えます。
- `PROXY` — Velocity経由でハブサーバーへ転送。転送先サーバー名は**Velocity側**の `config.yml`（`hub.server-name`）が決定し、Paper側からは指定できない設計です（改ざん・誤設定されたバックエンドが任意のサーバーへ転送させることを防ぐため、`HUB_TRANSFER_REQUEST` メッセージ自体に転送先サーバー名を含めていません）。`velocity.enabled: true` かつVelocity側の `orelia-serverutil-velocity` が稼働している必要があります。

### `/suadmin`

```
/suadmin reload
/suadmin setspawn
/suadmin worldsetup <world> [profile]
```

- `reload` — config/messagesを再読込。`ScoreboardModule`/`BelownameModule`/`TabListModule` はtitle・書式・更新間隔などをマネージャーへの再注入（`updateSettings`等）とスケジュールタスクの再作成で反映し、他プラグインが登録済みのプロバイダーは失われません。`CoreIntegrationModule` は `onEnable()` を再実行するだけの単純な方式（各`Core*Provider`の`getId()`が安定した定数のため、マップの該当エントリだけが上書きされる）。`VelocityBridgeModule` はチャンネルの再登録（`velocity.enabled`/`channel`の変更をリスタート無しで反映）を行います。
- `setspawn` — 実行者の現在地をそのワールドのスポーン地点に設定。
- `worldsetup <world> [profile]` — `config.yml` の `world-setup.profiles.<profile>.gamerules` を対象ワールドに一括適用（`profile`省略時は`default`）。任意のバニラGameRule名がそのままキーとして使えます。

## プレースホルダーシステム（`PlaceholderService`）

`config.yml` 内の `format`/`suffix-format`/`placeholder-format`/`lines` 系キー（`Core*Provider`、`AnnounceModule`、`TabListModule`のヘッダー/フッター、`ChatModule`のホバーツールチップ行）は全て `rpg.serverutil.paper.placeholder.PlaceholderService` という単一の共有インスタンス経由で解決され、各機能が個別に `String#replace` チェーンを持つことを避けています。`config.yml` 冒頭にトークン一覧のチートシートコメントがあり、以下の順で解決されます。

**常時使用可能:**

| トークン | 内容 |
|---|---|
| `{online}` | このサーバーの現在オンライン人数 |
| `{max_online}` | このサーバーのプレイヤー上限 |
| `{tps}` | このサーバーのTPS（1分平均、20.0でクランプ） |
| `{ping}` | **見ている側のプレイヤー**のping（ms） |
| `{world}` | 見ている側プレイヤーの現在ワールド名 |
| `{player}` / `{name}` | 見ている側プレイヤーの名前 |
| `{server}` | `server.name` で設定するこのサーバー自身の表示名（VelocityがこのサーバーをどうNamingしているかをPaper側は知らないため） |
| `{date}` | `yyyy-MM-dd` |
| `{time}` | `HH:mm:ss` |

**OreliaCore導入時のみ解決**（未導入時はリテラルテキストのまま）:

`{level}`・`{job}`（未選択なら空文字）・`{money}`（`1.5k`/`2m`/`3b`/`4t`のようなk/m/b/t表記）・`{health}`・`{max_health}`（バニラ体力ではなくOreliaCore独自のHPプール）

**OreliaExtra導入時のみ解決**（未導入時は空文字扱い）:

`{guild}`・`{guild_tag}`・`{party}`

最後に、PlaceholderAPIが導入されていれば `%...%` 記法もそのまま解決されます（`PlaceholderApiHook` が `me.clip.placeholderapi` を参照する唯一のクラスとして隔離されており、PlaceholderAPI未導入でも `NoClassDefFoundError` を起こしません）。

なお `belowname.title`（全プレイヤー共通の名札下ラベル）は per-viewer に解決される点に注意が必要です。`{health}` のような対象プレイヤー固有の値をここに書くと、全員に「見ている側自身の値」が表示されてしまいます（対象ごとに変わる値は `BelownameValueProvider` 側で解決）。

`chat.color-codes` はプレースホルダーとは別の仕組みで、`orelia.serverutil.chat.color` 権限（デフォルトop）を持つ送信者自身が入力した `&` カラーコード（レガシー・`&#RRGGBB`hex・OreliaCore独自の`&%<char>`コード）を `ColorUtil` でそのまま反映するものです。

## 公開API（`rpg.serverutil.api`、Bukkit ServicesManager経由）

他プラグインがサイドバー・タブリスト・名札下・チャットへ機能を差し込むための統合面です。いずれも `getId()` が安定したIDで登録・解除（`unregister*`）でき、優先度（`getPriority()`）が高い順に評価されます。実装例は `rpg.serverutil.paper.integration.CoreIntegrationModule` を参照してください。

- **`ScoreboardApi`** — `registerProvider(ScoreboardLineProvider)`/`unregisterProvider(String)`。`ScoreboardLineProvider#getLines(Player)` は優先度順にマージされ、同一プロバイダー内は返した順のまま表示されます。
- **`TabListApi`** — 2つの独立した仕組みを1つのAPIにまとめたもの。
  - `registerFormatter(TabListNameFormatter)` — `Optional<TabListEntry> format(Player)` で prefix/suffix/color（`TabListEntry`はrecord、`color`はnull可）を返す。最初に非空を返したフォーマッターが採用される。このTeamは名札（頭上表示）とタブリスト名の両方に同時に効く（Minecraftの仕様上、1エンティティは1Teamにしか所属できないため）。
  - `registerValueProvider(TabListValueProvider)` — タブリスト右側の値。名前装飾とは別の`Objective`（`PLAYER_LIST`スロット）を使い、viewerごとに異なる値を表示できる。
- **`BelownameApi`** — `registerProvider(BelownameValueProvider)`/`unregisterProvider(String)`。`getValue(Player target)` は**全viewer共通**の値（belowname値はMinecraft上viewerごとに変えられないため）。有効/無効フラグは存在せず、いずれかのプロバイダーが値を返した瞬間に自動表示、誰も返さなくなれば自動非表示になる（`BelownameManager`）。
- **`ChatApi`** — `registerProvider(ChatPlaceholderProvider)`/`unregisterProvider(String)`。`getPlaceholder(Player sender)` はチャット送信者名の横に表示する文字列（レベル・称号・ランク等）。

`ScoreboardManager`/`TabListManager`/`BelownameManager` は全て `rpg.serverutil.paper.util.BoardUtil.ensurePersonalBoard(Player)` を介して**同一の**per-playerスコアボードにオブジェクティブ/チームを重ね書きします（各自が新しいScoreboardをsetすると他機能の状態を消してしまうため）。新規にスコアボードベースの機能を追加する場合もBoardUtilの利用が前提です。

## Velocityブリッジ（`rpg.serverutil.common.protocol`、`*.bridge`）

Paper側とVelocity側は `orelia:serverutil` プラグインメッセージングチャンネル（`velocity.channel`で両側一致必須）と、自前バイナリフレーミングの `ProtocolCodec` で通信します。メッセージ種別は次の3系統です。

- **`HUB_TRANSFER_REQUEST`**（Paper→Velocity） — 転送先サーバー名を含まない。転送先はVelocity側`config.yml`の`hub.server-name`が決定する（改ざん耐性のための設計）。相関ID付きリクエスト/レスポンスとタイムアウト（`hub.proxy.request-timeout-seconds`）で構成される。
- **`HUB_TRANSFER_RESULT`**（Velocity→Paper） — 転送結果を、リクエストが来たのと同じ`ServerConnection`経由で返す。
- **`SERVER_SWITCH_NOTIFY`**（Velocity→Paper、転送先へ） — ベストエフォート。届かなくてもエラー扱いしない。`ServerSwitchListener.onServerConnected`から無条件に送信され、`JoinMessageModule`の参加ブロードキャストに使われる。
- **`SERVER_SWITCH_LEAVE_NOTIFY`**（Velocity→Paper、転送元へ） — `SERVER_SWITCH_NOTIFY`と同じペイロード形状（ワイヤータグのみ異なる）。退出プレイヤー自身は既に転送元サーバーから切断済みのため、転送元に残っている**他の**オンラインプレイヤーを介してVelocityが中継する。誰も残っていなければ送信されない（見せる相手がいないため）。

`SERVER_SWITCH_(LEAVE_)NOTIFY` はネットワーク越しに非同期で届く一方、`PlayerJoinEvent`/`PlayerQuitEvent`はPaper自身のログインシーケンスから発火するため、どちらが先に届くか保証がありません。`awaitArrival`/`awaitDeparture`（`VelocityBridgeModule`）は、先に届いた側が自分を登録（`pendingArrivals`/`pendingDepartures`、または`arrivalWaiters`/`departureWaiters`）しておくことで、後から届いた側が即座にブロードキャストを確定させる仕組みです。`join.server-switch-wait-ticks`（既定20tick）は通知が遅延・消失した場合のみ効く最悪ケースのタイムアウトであり、通常はそれより早く確定します。

サーバー切り替え時のjoin/leaveメッセージは、通常のjoin/quitメッセージの代わりに転送元・転送先それぞれのサーバーへローカル表示されます（Velocity全体へのブロードキャストではありません）。初回ログインや、切替通知が届かないまま完全切断した場合は、通常のjoin/quitメッセージにフォールバックします。プラグインメッセージの送信にはオンラインの`Player`が運び手として必要で、コンソールから直接送る経路はありません。

## 管理者向けHealthCheck

`admin-healthcheck.enabled: true`（既定）の場合、op権限のプレイヤーがjoinするとTPS/オンライン人数のサマリーを表示します。`core-integration.enabled: true`（既定）の場合はOreliaCore/OreliaWorld/OreliaExtra/OreliaDebugそれぞれの導入状況・バージョンも1行で表示されます。

## config.yml 主要セクション

`server`, `velocity`, `hub`, `world-setup.profiles.*`, `scoreboard`（`core-lines`含む）, `tablist`（`name-color`/`value`含む）, `belowname`（`core-format`含む）, `chat`（`tooltip`/`color-codes`/`core-placeholder`含む）, `announce`, `admin-healthcheck`, `join`, `core-integration`。`core-integration.enabled` はOreliaCore連携機能全体（サイドバー行・タブリスト名色/値・名札下・チャットプレースホルダー）のマスタースイッチで、これがfalseだと各セクション個別の`enabled`に関わらず何も動きません。

## 依存関係まとめ

- **orelia-core**（任意・softdepend） — 未導入でも起動する。導入時は`CoreIntegrationModule`がレベル/職業/所持金/HPを各API経由でサイドバー・タブリスト・名札下・チャットへ自動反映する。
- **OreliaExtra**（任意・softdepend） — 未導入でも起動する。導入時は`{guild}`/`{guild_tag}`/`{party}`プレースホルダーが解決される。
- **Velocity + orelia-serverutil(velocity)**（任意） — `hub.mode: PROXY`やサーバー切り替え演出に必要。paper単体では`hub.mode: TELEPORT`のみ利用可能。
