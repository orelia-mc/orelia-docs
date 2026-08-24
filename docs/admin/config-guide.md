# 設定ファイルの書き方(基本・config.yml)

## 設定ファイルの基本

- すべての設定ファイルは `plugins/OreliaCore/` フォルダに置かれます(サーバー初回起動時に自動で作られます)。
- YAMLファイルはテキストエディタで直接編集します。編集後は `/oladmin reload` を実行すれば、サーバーを再起動せずに反映されます。[`/oladmin reload`](core-commands.md#reload) は、そのリロードで実際に変更・追加・削除された項目を一覧表示してくれるので、編集内容が意図通り反映されたかその場で確認できます。
- 各ファイルの先頭には `config-version:` という行があります。新しい設定項目がプラグインのアップデートで追加されたとき、この番号を比較して**既存ファイルへ自動で追記**されます(あなたが編集済みの値が上書きされることはありません)。この番号は手動で触る必要はありません。
- 色付き文字は `&%` に続けて1文字のカラーコード(`&%a`=緑、`&%c`=赤、`&%e`=黄、`&%7`=灰色など)で指定します。バニラの `&`カラーコードと同じ体系です。

## `config.yml` の構成

`config.yml` はプラグイン全体に関わる横断的な設定です。主なセクションは以下のとおりです。

### `chat`

```yaml
chat:
  mute:
    enabled: true
```

`/chat mute` コマンド自体の有効/無効を切り替えます。`false` にすると、既存のミュート設定も含めて `/chat mute` が完全に無効化されます。

### `status`(レベル・ステータス成長)

```yaml
status:
  regen:
    hp-percent-per-tick: 0.5
    sp-percent-per-tick: 1.0
    period-ticks: 100
  leveling:
    exp-per-level: 100
    max-level: 80
  growth:
    HP: { base: 100 }
    SP: { base: 50 }
    ATK: { base: 5 }
    DEF: { base: 5 }
    CRT: { base: 10 }
    CRT_DMG: { base: 50 }
    SPD: { base: 5, per-level: 1 }
```

- `regen` — HP/SPの自動回復速度(`period-ticks`ごとに最大値の何%回復するか)。
- `leveling.exp-per-level` — 1レベル上げるのに必要な経験値。`max-level` — レベル上限。
- `growth` — 各ステータスの基礎値。HP/SP/ATK/DEFは`stat-scaling.growth-rate`(後述)に従って指数的に成長し、SPDは`per-level`ずつ一定量ずつ増えます。CRT/CRT_DMGはレベルに関わらず固定です(レリック・アクセサリーのボーナスのみで変化)。
- ステータス成長の仕組み自体は [ゲーム内ロジック](game-logic.md) を参照してください。

### `stat-scaling`

```yaml
stat-scaling:
  growth-rate:
    HP: 1.045
    SP: 1.04
    ATK: 1.035
    DEF: 1.03
```

キャラクターとモンスター(スポーンポイントで目安レベルを指定した場合)が共有する、レベルごとの成長倍率です。詳しくは[ゲーム内ロジック](game-logic.md)を参照してください。

### `weapon-level`

```yaml
weapon-level:
  attack-power-factor: 0.05
  initial-cap: 5
  tier-step: 10
```

武器レベル(強化とは別枠、[遊び方: 装備・アイテム](../play/items.md)参照)の上昇量と、プレイヤーレベルに応じた上限を決めます。

### `combat`

```yaml
combat:
  scaled-health:
    vanilla-cap: 20
  damage-display:
    enabled: true
    duration-ticks: 20
    normal-color: "&%f"
    crit-color: "&%6"
    crit-scale: 1.3
```

ダメージ数値の表示演出と、バニラ体力バーの見た目調整(`vanilla-cap`)です。数値そのものの計算式には影響しません。

### `action-bar`

```yaml
action-bar:
  enabled: true
  period-ticks: 20
  format: "&%eLv.{level} {exp_bar} &%7| &%c♥ {hp}/{max_hp} &%7| &%b✦ {sp}/{max_sp} &%7| &%e⚔ {atk}"
```

常時表示されるアクションバーのHUD文言です。`{level}`・`{exp_bar}`・`{hp}`/`{max_hp}`・`{sp}`/`{max_sp}`・`{atk}` が使えます。

### `town-detection`

```yaml
town-detection:
  enabled: true
  town-regions:
    - town1_areaA
    - town1_areaB
```

WorldGuardのリージョンIDを町として登録すると、その中ではOrelia側のモンスター自動湧きが発生しなくなります。WorldGuardが導入されていない場合は何も起きません。1つの町が離れた複数エリアにまたがる場合は、エリアごとに別々のWorldGuardリージョンを作り、そのIDを全部列挙してください。

### `monster`

```yaml
monster:
  disable-vanilla-hostile-spawning: true
  sun-immunity:
    enabled: true
  health-bar:
    enabled: true
    length: 10
    format: "{name} &%7[{bar}&%7] &%f{current}/{max}"
  target-level-bonus:
    HP: 1.02
    ATK: 1.015
    DEF: 1.012
```

- `disable-vanilla-hostile-spawning` — バニラの自然湧き/スポナー湧きを無効化し、モンスターは全て[スポーンポイント](core-commands.md#spawnpoint)経由にします。
- `target-level-bonus` — 目安レベル付きモンスターにのみ乗る追加倍率。詳しくは[ゲーム内ロジック](game-logic.md)を参照。

### `quest` / `dungeon` / `party` / `friend` / `guild` / `mail` / `auction` / `trade`

各機能の細かい調整値です。多くは名前から用途が分かるようになっています(例: `dungeon.relic-drop-min/max` はダンジョンボス撃破時のレリックドロップ数、`party.max-size` はパーティー人数上限)。`notify-sound` ブロック(`party`・`guild`・`mail`・`trade`に共通)は招待/取引申込などの通知音を制御します。

```yaml
party:
  max-size: 6
  notify-sound:
    enabled: true
    name: ENTITY_EXPERIENCE_ORB_PICKUP
    volume: 1.0
    pitch: 1.0
```

`name` は `org.bukkit.Sound` の定数名です。無効な名前を指定した場合はサウンドだけ無視され、機能自体は動作します。
