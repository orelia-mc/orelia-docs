# 設定ファイルの書き方(コンテンツ定義)

`config.yml`([前のページ](config-guide.md))以外の設定ファイルは、いずれも「IDをキーにしたマップ」の形でゲーム内コンテンツ(クエスト・NPC・モンスターなど)を定義します。新しいエントリを追加するにはコードの変更は不要で、ファイルに新しいキーを足して `/oladmin reload` するだけです。ファイル名・トップレベルのキー名は固定です。

## クエスト・NPC・会話

### `quests.yml`

```yaml
quests:
  slime_cull:
    name: "スライム討伐"
    type: MAIN                 # MAIN | SUB | DAILY | WEEKLY | EVENT
    required-level: 1
    repeatable: false
    party-only: false
    prerequisite-quests: []
    description:
      - "&%7森のスライムを5体討伐しよう。"
    objectives:
      obj_1:
        type: KILL_MONSTER      # KILL_MONSTER | COLLECT_ITEM | DELIVER_ITEM | REACH_LOCATION | TALK_NPC | KILL_BOSS | CLEAR_DUNGEON
        target-id: forest_slime  # monsters.yml/bosses.yml/npc.yml/dungeons.yml のID、またはアイテムID
        amount: 5
    reward:
      exp: 50
      money: 20
      skill-points: 1
```

- `target-id` は目標の種類によって参照先が変わります(`REACH_LOCATION`だけは`target-id`の代わりに`world`/`x`/`y`/`z`/`radius`を使います)。
- `repeatable: true` のクエストは `cooldown-hours` で再受注までの待機時間を設定できます(省略時は即座に再受注可能)。
- `prerequisite-quests` に別のクエストIDを指定すると、それを達成するまで受注できません。
- `party-only: true` はパーティー所属時のみ受注可能にします。
- `available-hour-start`/`available-hour-end`(0〜23、省略時は常に受注可能)で時間帯限定クエストが作れます。
- `start-dialogue-id`/`complete-dialogue-id` に `dialogues.yml` のツリーIDを指定すると、受注/報告時に会話演出が流れます(任意)。

### `npc.yml`

```yaml
npcs:
  weapon_merchant:
    name: "武器商人ガレット"
    type: WEAPON_SHOP    # WEAPON_SHOP | ARMOR_SHOP | ACCESSORY_SHOP | QUEST_RECEPTIONIST |
                          # JOB_CHANGE | ENHANCEMENT | WAREHOUSE | GUILD_RECEPTIONIST |
                          # WEAPON_LEVELUP | RELIC_UPGRADE
```

ショップNPCの `shop-stock` には `kind: WEAPON|ACCESSORY|RELIC|VANILLA` のアイテムを並べられます。`RELIC_UPGRADE` タイプは追加設定不要で、`/ol relic upgrade` と同じ画面を開きます。NPCの実際の設置は [`/oladmin npc create`](core-commands.md#npc) で行います(このファイルは定義のみで、自動では設置されません)。

### `dialogues.yml`

会話ツリーです。`entry-points` は上から順に評価され、プレイヤーが条件(`required-flag`)を満たす最初のものが使われます。選択肢のない会話はそのまま次のノードへ進むか、会話を終了します。クエストの受注/報告演出もここに登録します。

### `story.yml`

章立てのストーリー進行です。各章はクエスト報酬や会話の選択肢などで得られる「フラグ」を全て満たすと解放されます(`order` で表示順、`required-flags` で解放条件)。

### `cutscenes.yml`

`CAMERA`/`MESSAGE`/`EFFECT`/`TITLE` の4種類のステップを、`delay-ticks`(再生開始からの相対時間、20 tick=1秒)で並べて演出を組み立てます。

### `events.yml`

期間限定/季節イベントの定義です。`recurring: true` は `--MM-DD` 形式の毎年繰り返す期間(季節イベント向け)、`recurring: false` はISO-8601形式の一回限りの期間(限定イベント向け)を使います。

## モンスター・ダンジョン

### `monsters.yml`

```yaml
monsters:
  forest_slime:
    entity-type: SLIME
    ai-type: AGGRESSIVE      # PASSIVE | AGGRESSIVE | RANGED
    weakness: NONE           # NONE | FIRE | WATER | EARTH | WIND | LIGHT | DARK
    hp: 50
    attack: 5
    defense: 2
    drops:
      slime_ball:
        vanilla-material: SLIME_BALL
        chance: 0.5
```

- `weakness` に指定した属性の武器で攻撃すると、常に×1.5のダメージが入ります。
- `abilities`(任意)で `AOE_SLAM`(範囲攻撃)/`FIREBALL_BARRAGE`(火球連射)のような周期的な特殊攻撃を追加できます。`damage` はそのモンスター自身の攻撃力に対する**倍率**です。
- `crit-rate`/`crit-multiplier` でこのモンスターが会心攻撃をする確率と倍率を設定できます(既定は会心なし)。

### `bosses.yml`

```yaml
bosses:
  goblin_king_boss:
    monster-id: goblin_raider   # 土台にするmonsters.ymlのID
    phases: [...]               # HP割合ごとの段階演出
    enrage-multiplier: 1.5
```

既存の `monsters.yml` エントリを土台に、HP割合ごとのフェーズ演出・狂暴化倍率・特殊攻撃を追加したものです。湧き方・攻守・撃破報酬は土台のモンスター側の設定に従います。

### `dungeons.yml`

```yaml
dungeons:
  goblin_cave:
    name: "ゴブリンの洞窟"
    type: NORMAL          # NORMAL | STORY | RAID | SOLO | PARTY
    min-party-size: 1
    max-party-size: 4
    arenas:
      - { world: world, x: 100, y: 64, z: 100 }
      - { world: world, x: 150, y: 64, z: 100 }
    reward-exp: 150
    reward-money: 80
    enemies:
      goblin_raider: 5
      skeletal_archer: 3
    boss-id: goblin_king_boss   # 省略可(ボスなしダンジョン)
    time-limit-seconds: 300
```

`arenas` のエントリ数がそのまま同時挑戦可能パーティー数になります。ダンジョンの詳しい挙動(難易度・スケーリング・クリア判定)は [ゲーム内ロジック: ダンジョンのロジック](game-logic.md) を参照してください。ダンジョン開放ブロックの設置は [`/oladmin dungeonblock set`](core-commands.md#dungeonblock) で行います。

## アイテム・スキル・職業

### `items.yml`

```yaml
weapons:
  iron_sword:
    weapon-type: SWORD    # SWORD | SPEAR | AXE | BOW | PICKAXE | HOE | HATCHET | WAND
    rarity: COMMON         # COMMON | UNCOMMON | RARE | EPIC | LEGENDARY
    element: NONE
    required-job: FENCER   # 空欄なら誰でも装備可
    attack-power: 20
    skill-slot-count: 1
```

新しい武器を追加するにはコード変更不要で、`weapons:` に新しいキーを足すだけです。

### `skills.yml`

```yaml
skills:
  power_strike:
    weapon-type: SWORD
    executor-type: MELEE_CONE   # MELEE_CONE | MELEE_AOE | DASH_STRIKE | ARROW_VOLLEY | EXPLOSIVE_ARROW
    damage-multiplier: 1.5
    sp-cost: 20
    cooldown-seconds: 8
```

`executor-type` がスキルの挙動の型を決めます。

### `jobs.yml`

```yaml
jobs:
  FENCER:
    display-name: "フェンサー"
    allowed-weapons: [SWORD]
    passive-bonus:
      ATK: 2
```

`allowed-weapons` で装備できる武器種を制限し、`passive-bonus` は転職しているだけで常時付与される固定ステータス加算です。

### `accessories.yml` / `relics.yml`

- `accessories.yml` — ショップで買える固定ステータスのアクセサリー(`type: CHARM|RING|NECKLACE|WING`)。
- `relics.yml` — ダンジョンボスがドロップするランダム生成アクセサリーの生成ルール(部位ごとのメインステータス候補・サブステータスの伸び幅・ダンジョンセットボーナス)。詳しくは[遊び方: 装備・アイテム](../play/items.md)を参照。

### `effects.yml`

パーティクル/サウンドの組み合わせを名前付きで再利用できるようにする定義です。

### `gui.yml` / `crafting.yml` / `fishing.yml` / `gathering.yml`

- `gui.yml` — 各画面のタイトル文言の上書き(未指定なら既定の日本語タイトル)。
- `crafting.yml` — `/ol craft` の合成レシピ(結果は武器のみ、素材はバニラのMaterial名)。
- `fishing.yml` — 釣り人の待ち時間・釣れるアイテムの抽選テーブル(エリア/ワールドごと)。
- `gathering.yml` — 採掘/伐採/農業のクールダウン・経験値・再生成時間。

## 実績・住居・ペット・乗り物

### `achievements.yml`

```yaml
achievements:
  reach_level_10:
    name: "駆け出し冒険者"
    description: "レベル10に到達する"
    condition-type: REACH_LEVEL   # REACH_LEVEL | COMPLETE_QUEST | MONEY_BALANCE
    category: "成長"
```

### `housing.yml` / `mounts.yml` / `pets.yml`

いずれも「購入できるものの一覧」を定義するシンプルな構造です(名前・価格・座標や見た目などの基本情報)。既存のエントリを見本にすれば新規追加は容易です。
