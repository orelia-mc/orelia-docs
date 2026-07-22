# Monster（モンスター）の動作確認

対象: `rpg.monster` + `rpg.monster.spawnpoint`（[Monster モジュール仕様](../core/monster.md)）。バニラエンティティをOreliaのモンスターテンプレートとしてスポーンさせ、被弾時にステータスを適用、撃破時に経験値/お金/ドロップを報酬として与えます。

## 前提条件

`monsters.yml` 標準定義：

| ID | 表示名 | エンティティ | HP | 攻撃力 | 防御 | 弱点属性 | 経験値 | お金 |
|---|---|---|---|---|---|---|---|---|
| `forest_slime` | 森のスライム | SLIME | 15 | 2 | 0 | FIRE | 5 | 1〜3 |
| `goblin_raider` | ゴブリンの略奪者 | ZOMBIE | 40 | 5 | 2 | なし | 15 | 5〜12 |
| `skeletal_archer` | 骸骨の弓兵 | SKELETON | 30 | 4 | 1 | なし | 18 | 6〜14 |
| `goblin_king` | ゴブリンキング | ZOMBIE | 400 | 12 | 5 | なし | 200 | 100〜250 |
| `flame_lord` | 業火の王ブレイズロード | BLAZE | 600 | 15 | 8 | なし | 800 | 300〜600 |

`goblin_raider`（`war_cry_slam`, AOE_SLAM, damage 8, radius 3, cooldown 10秒）と `skeletal_archer`（`numbing_volley`, AOE_SLAM, damage 6, radius 4, cooldown 12秒）には `abilities` が設定されています（下記5.）。

## 1. スポーンコマンドの確認

```
/oladmin spawn forest_slime
```

期待結果: 自分の足元にSLIMEエンティティがスポーンし、`Attribute.MAX_HEALTH` が `min(hp, config.yml: combat.scaled-health.vanilla-cap)`（デフォルト1024。`forest_slime` は `hp=15` なのでそのまま15）に設定されている（バニラのステータスバーやスコアボードで確認できれば実測、できなければ後述のダメージテストで間接確認）。`flame_lord`（hp=600）のように `vanilla-cap` 未満のモンスターは今のところ全てバニラ体力=スケール済みHPと一致するが、`vanilla-cap` を意図的に低く設定（例: 100）した状態でHPの大きいモンスターをスポーンさせ、バニラ体力が頭打ちになる一方でHPバー（後述）はスケール済みの実数値を表示することも確認しておくとよい。

- 存在しないID（`/oladmin spawn nonexistent_id`）はエラーになり、何もスポーンしないこと。
- `AGGRESSIVE`（`goblin_raider`, `goblin_king`, `flame_lord`）はプレイヤーに近づき攻撃してくる、`PASSIVE` は襲ってこない、`RANGED`（`skeletal_archer`）は離れた位置から攻撃してくることを `ai-type` どおりに確認する。

## 2. 被弾時のダメージ計算の確認（`CombatDamageListener`）

被ダメージ処理は `rpg.monster.listener.CombatDamageListener`（`LOW` 優先度、`monster` モジュール側で登録）に一本化されています。旧 `MonsterCombatListener`/`WeaponUseListener`/`CombatStatusListener` の3リスナー構成は撤去済みです。計算順序は `rpg.status.combat.DamageFormula.compute`：base attack power → ATK% → DEF軽減 → クリティカル → 属性弱点。

1. `goblin_raider`（攻撃力5）をスポーンさせ、プレイヤーが攻撃を受けたときのダメージがバニラのゾンビダメージではなく5前後（[Status のダメージ計算式](status.md)の軽減も加味）であることを確認する。
2. モンスター側が被弾した場合、`defense/(defense+100)` の軽減がプレイヤーの攻撃ダメージに適用されること（`defense=5` の `goblin_king` なら約4.8%軽減）を確認する。
3. `monsters.yml` に `crit-rate`/`crit-multiplier` を設定したモンスター（標準定義には無いため検証用に一時追加するか、`/ol item give` 等で武器のクリティカル率と合わせて確認）でクリティカルが発生した場合、攻撃者に一時的なメタデータが立ち、直後の [ダメージ数値表示](#3-hp) で色/拡大表示が変わることを確認する。

## 2.5. 弱点属性（elemental weakness）の確認

`monsters.yml` の `weakness`（省略時 `NONE`）は、攻撃者の手持ち武器の `element`（`items.yml`）が一致した場合にのみ、DEF軽減後・クリティカル判定後のダメージへ **1.5倍**の固定倍率を追加で乗せます（`DamageFormula.applyElementalWeakness` / `CombatDamageListener.isWeaknessHit`）。標準定義では `forest_slime` だけが `weakness: FIRE` を持ちます。

1. `element: NONE` の武器（例: `wooden_training_sword`）で `forest_slime` を攻撃し、通常ダメージ（弱点倍率なし）であることを確認する。
2. `element: FIRE` の武器（例: `flame_longsword`）で同じ `forest_slime` を攻撃し、同条件の1と比べてダメージが1.5倍になっていることを確認する。
3. `weakness` が設定されていない他のモンスター（`goblin_raider` 等）を `flame_longsword` で攻撃しても倍率が乗らない（`NONE` は弱点ヒット扱いにならない）ことを確認する。
4. 弱点倍率は防御軽減・クリティカルの**後**に乗算される（`DamageFormula.compute` の最終ステップ）ため、DEFが高いモンスターに弱点武器で攻撃した場合の最終値を電卓で検算しておくと良い。

## 3. HPバー・フローティングダメージ数値の確認

1. モンスターへ攻撃するたびに、ネームタグのHPバー（`config.yml: monster.health-bar.format` のデフォルト `"{name} [{bar}] {current}/{max}"` 相当、`length=10` セグメント）が現在HP/最大HPの割合どおりに減っていくことを確認する（`MonsterHealthBarListener`、`MONITOR` 優先度）。
2. `monster.health-bar.enabled: false` にすると、バー無しの通常名前表示に戻ることを確認する（切り戻し確認）。
3. 攻撃が命中するたびに、命中位置付近にダメージ数値の表示エンティティが現れ、`config.yml: combat.damage-display.duration-ticks`（デフォルト20tick）の間、`gravity-per-tick`（デフォルト0.02）で徐々に落下速度を増しながら消えることを確認する（`DamageDisplayListener`、`MONITOR` 優先度）。
4. クリティカルヒット時は `crit-color`/`crit-scale`（デフォルト1.3倍）で通常時と見た目が変わることを確認する。
5. `hp` がバニラ体力の範囲を超えるモンスター（`goblin_king` hp=400等）でも、表示される数値はバニラ換算後の小さい値ではなく、スケール済みの実ダメージ量（例: 素手で数十〜百のダメージ表記）になっていることを確認する。

## 4. 撃破報酬の確認（`MonsterDropService.rewardKiller`）

1. プレイヤーで `forest_slime` を倒す。
2. 経験値が5加算されること（`/ol status` の経験値表示、または直後のレベルアップ有無で確認）。
3. お金が `money-min`〜`money-max`（1〜3）の範囲でランダムに加算されること（[Economy の確認手順](economy.md)と合わせて残高を確認）。
4. `slime_ball`（バニラ素材、確率60%）がその確率通り、複数回撃破を試行して概ね60%前後の頻度でドロップすることを確認する（1回だけでは判定できないため、10〜20体単位で試すことを推奨）。
5. `goblin_raider` を倒し、低確率（5%）の `wooden_training_sword` ドロップが `itemManager.createWeapon` 経由で生成された正規のOrelia武器（PDC付き）であることを確認する。
6. **プレイヤーによる撃破の場合のみ**バニラの標準ドロップ/経験値がクリアされ、`rewardKiller` の内容だけが得られること（`MonsterDeathListener` の分岐）。モンスター同士の共食い等、プレイヤー以外による撃破ではこの上書きが起きないことも余力があれば確認する。

## 5. 通常モンスターの能動アビリティの確認（`MonsterAbilityCastService`）

1. `goblin_raider`（`war_cry_slam`: AOE_SLAM, damage 8, radius 3, cooldown 10秒）を `/oladmin spawn`（または後述のスポーンポイント経由）でスポーンし、`AGGRO_RANGE=24.0` 以内に留まる。20tickごとの `tick()` により、クールダウン明けにアナウンスメッセージ（`&%eゴブリンの略奪者が雄叫びを上げた！`）と共に半径3ブロック以内のプレイヤーへ直接ダメージが入ることを確認する。
2. `skeletal_archer`（`numbing_volley`: damage 6, radius 4, cooldown 12秒）でも同様に確認する。
3. 同じ個体が同tickで複数のアビリティを同時発動しない（最大1つ）ことを確認する。
4. クールダウン中は再発動しない（`war_cry_slam` なら10秒、`numbing_volley` なら12秒）ことを確認する。
5. `AGGRO_RANGE` より離れるとアビリティが発動しなくなることを確認する。
6. アビリティのダメージは `CombatDamageListener` を経由しない（`player.damage(amount)` を直接呼ぶ）ため、モンスターの通常攻撃力（`attack-power`）の影響を受けず設定値どおりの固定ダメージになることを、モンスターの通常攻撃（噛み付き等）と比較して確認する。
7. **`goblin_king`/`flame_lord` に `abilities` を後付けで設定してテストする場合**、`/oladmin spawn goblin_king`（通常湧き経路）ではアビリティが発動するが、`bosses.yml` 経由の `/oladmin spawnboss goblin_king_boss` で湧かせた場合は `BossAbilityCastService` 側のみが動作し、`MonsterAbilityCastService` には登録されない（[Boss の確認手順](boss.md)参照）ことを確認する。

## 6. スポーンポイントの確認

```
/oladmin spawnpoint add goblin_raider 30 3
/oladmin spawnpoint list
```

期待結果:

1. 現在地にスポーンポイントが登録され、`spawnpoint list` に表示される。
2. `intervalSeconds=30`（省略時デフォルト）、`maxAlive=3`（省略時デフォルト）で登録されていることを一覧で確認。
3. 20tickごとの `tick()` により、30秒経過後に自動で `goblin_raider` がスポーンし始める。
4. 同時生存数が3体に達すると、それ以上はスポーンしない（`aliveCount >= maxAlive` でスキップ）ことを確認する。
5. スポーンされた個体を倒すと `onEntityRemoved` でスロットが解放され、次の間隔でまた湧くようになることを確認する。

```
/oladmin spawnpoint remove <id>
```

削除後、そのスポーンポイントから新たな湧きが起きなくなることを確認する。

## 7. バニラ自然湧きの無効化確認

`config.yml: monster.disable-vanilla-hostile-spawning`（デフォルト `true`）。

1. モンスタースポナーや自然湧き（`CreatureSpawnEvent` の理由が `NATURAL`/`SPAWNER`）で敵対的バニラMobが湧かなくなっていることを暗い場所等で確認する。
2. `false` に変更して `/oladmin reload` した場合、通常のバニラ自然湧きが復活することを確認する（切り戻し確認）。
3. Oreliaモンスターのスポーン（`/oladmin spawn` やスポーンポイント経由）はこの設定に関係なく常に動作することを確認する。

## 異常系・エッジケース

- 密集した場所で複数のモンスターを同時に倒し、報酬計算（経験値・お金・ドロップ）が個体ごとに正しく独立して処理されること（合算されすぎたり、取りこぼしたりしないこと）。
- スポーンポイントを世界の異なる場所に複数登録し、それぞれ独立してカウント・間隔管理されること。
