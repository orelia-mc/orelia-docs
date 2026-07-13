# Skill（武器スキル/技）の動作確認

対象: `rpg.skill`（[Skill モジュール仕様](../core/skill.md)）。`skills.yml` の定義を「executor」アーキタイプが実行し、武器にスキルをソケットし、スキルポイントで習得・レベルアップします。

!!! danger "既知の未接続ギャップ: SkillGuiScreen への導線が無い"
    技のソケット（`SkillSocketService.socket`）・習得/レベルアップ（`SkillProgressService.upgradeSkill`）は `SkillGuiScreen` のクリックハンドラからのみ呼ばれますが、このドキュメント作成時点のコードベースには **`SkillGuiScreen` を開くプレイヤー向けの導線が一切ありません**。
    
    - `orelia-core` 側: `/ol` に登録されている player コマンドは `item` / `job` / `status` の3つのみで、スキル画面を開くコマンドはありません。
    - `orelia-world` 側: `NpcType` は `WEAPON_SHOP, ARMOR_SHOP, ACCESSORY_SHOP, QUEST_RECEPTIONIST, JOB_CHANGE, ENHANCEMENT, WAREHOUSE, GUILD_RECEPTIONIST` の8種のみで、スキル画面を開くNPC種別が存在しません。
    - `GuiApi.openSkill(Player)` というAPIメソッド自体は `rpg.api` に存在しますが、`orelia-core`/`orelia-world` のどちらのコードからも呼び出されていません。
    
    これは実装バグではなく「未接続」の状態です。ドキュメントを直すために存在しない導線を書き足すことはせず、この節では**検証時の回避策**を示します。恒久対応（NPCタイプ追加やコマンド追加）が入った場合は、この節を実際の導線に置き換えてください。

## 検証のための回避策

以下のいずれかの方法で `SkillGuiScreen` に到達してください。

1. **一時的な検証用コマンド/NPCを足す**: `GuiApi.openSkill(player)` を呼ぶだけの1行コマンド、または `npc.yml` に暫定で新しい `NpcType`（要コード変更）やテスト用リスナーを追加し、動作確認後に取り除く。
2. **サービスを直接叩く小さな検証プラグイン**: `SkillSocketService.socket(ItemStack, String skillId, int maxSlots)` は純粋なアイテム操作（PDC文字列 `"socketed_skills"` への書き込み）なので、プレイヤー無しでも呼び出せます。手持ちの武器の `skill-slot-count`（`items.yml`）を `maxSlots` として渡し、対象スキルIDをソケットします。
3. **`SkillProgressService.grantSkillPoints`/`upgradeSkill`** を検証プラグインから直接呼び出し、スキルポイント付与→習得（`upgradeSkill` はSP1消費、+1レベル、`maxLevel`で頭打ち）を行う。

いずれの方法でも、以下のGUI由来の仕様は個別に確認しておいてください（`SkillGuiScreen` のロジック）：

- 識別可能なOrelia武器を手に持っていない状態でGUIを開くとBARRIER表示になる。
- 手に持っている武器の `weapon-type` に対応するスキルだけが一覧表示される（例: SWORD装備中なら `slash`/`spin_slash`/`iaijutsu`/`cross_slash` の4つ）。
- 一覧はENCHANTED_BOOKアイコンで、現在レベル/最大レベル/SPコスト/クールダウンが表示される。
- 左クリックでアップグレード（`upgradeSkill`）、右クリックでソケット（`socket`）。

## 1. ソケット・習得ができていることの確認

回避策でソケット・習得を行った後、武器を実際にインベントリへ持たせて:

1. 武器のPDCに `socketed_skills`（`;` 区切り）でスキルIDが入っていることを確認する（`SkillSocketService.getSocketedSkills`/`hasSkill` 相当の状態）。
2. `PlayerSkillComponent.skillLevels` にそのスキルのレベルが1以上入っていること（`SkillCastService.cast` が `NOT_LEARNED` を返さないことで間接確認できる）。

## 2. スキル発動（右クリック等）の確認

`SkillActivationListener` のトリガー：

- 通常の右クリック = スロット0のスキル
- シフト+右クリック = スロット1のスキル
- オフハンド切替キー = スロット2のスキル（弓でも同様に動作）

各武器種で1つずつ、スロット0に技をソケット・習得した状態で発動を確認します。

| 武器種 | 例 | executor-type | 確認ポイント |
|---|---|---|---|
| SWORD | 斬撃 `slash` | MELEE_CONE | 前方コーン範囲内の敵にのみダメージ、`SWEEP_ATTACK`パーティクル |
| SWORD | 回転斬り `spin_slash` | MELEE_AOE | 自分中心・半径3.5の全方位にダメージ |
| SWORD | 居合 `iaijutsu` | DASH_STRIKE | 前方に踏み込み、最初に当たった敵にダメージ（ダメージ倍率2.2倍が突出して大きい） |
| SPEAR | 突進 `charge_thrust` | DASH_STRIKE | 前方へダッシュ、ノックバック0.7と大きめ |
| AXE | バーサーク `berserk` | MELEE_AOE | 半径4.0、クールダウン15秒と長め |
| BOW | パワーショット `power_shot` | ARROW_VOLLEY | 単発・高精度の矢（`radius: 0`） |
| BOW | マルチショット `multi_shot` | ARROW_VOLLEY | 3本が`radius: 20`度に拡散して発射される |
| BOW | 爆裂矢 `explosive_arrow` | EXPLOSIVE_ARROW | 着弾時に半径3.0で爆発ダメージ |

各テストで以下を確認します。

1. `SkillCastService.cast` が成功する（`CastFailure` が返らない）こと。
2. 発動中は通常の武器攻撃ダメージ処理が抑制される（`orelia_skill_active` メタデータによる抑制、`WeaponUseListener` 側の仕様）こと。
3. `effect-particle`（表に記載）が視覚的に発火すること。
4. `damage-multiplier` に応じたダメージが通常攻撃より明確に大きいこと（例: `slash` は1.4倍、`iaijutsu` は2.2倍）。
5. `scaledDamageMultiplier(level) = damageMultiplier * (1 + 0.1 * max(0, level - 1))` により、レベルを上げるとダメージがわずかに増えること（レベル1→5で最大40%増）。

## 3. `CastFailure` 各パターンの確認

`SkillCastService.cast` は 武器種一致 → ソケット有無 → 習得レベル>0 → クールダウン → SPコスト の順にチェックします。意図的に条件を崩し、それぞれ正しく弾かれることを確認します。

| 状況 | 期待される `CastFailure` |
|---|---|
| 手に何も持たず発動を試みる、または非Orelia武器 | `WRONG_WEAPON`（または `UNKNOWN_SKILL`） |
| 対応武器だがそのスキルをソケットしていない | `NOT_SOCKETED` |
| ソケット済みだが未習得（レベル0） | `NOT_LEARNED` |
| クールダウン中に再発動 | `ON_COOLDOWN` |
| SPが足りない状態（`tryConsumeSp` 失敗） | `NOT_ENOUGH_SP` |
| `executor-type` に対応する `SkillExecutor` が未登録 | `NO_EXECUTOR`（通常発生しないはずなので、発生したら `SkillExecutorRegistry` の登録漏れを疑う） |

各失敗時、プレイヤーへ適切なフィードバック（メッセージ・効果音等、実装があれば）が返り、SP/クールダウンが誤って消費されないことも確認してください。

## 4. クールダウン・SP消費の確認

1. スキルを発動し、`cooldown-seconds` 未満の間隔で再発動を試みて `ON_COOLDOWN` になることを確認する。
2. `cooldown-seconds` 経過後は再び発動できることを確認する。
3. 発動のたびに `sp-cost` 分だけSPが減っていることを `/ol status` で確認する（[Status の確認手順](status.md)のSP自然回復と合わせて、回復より消費が早いと連発できなくなることも確認）。
4. `PlayerSkillComponent.cooldownExpiry` は永続化されない仕様のため、発動直後にサーバーを再起動/再参加すると、クールダウンがリセットされて即座に再発動できてしまうことを確認する（バグではなく仕様）。

## 5. 矢系スキル固有の確認

- **`multi_shot`**（`ArrowVolleyExecutor`）: メタデータ `orelia_skill_arrow_multiplier` が矢に付与され、`ArrowSkillDamageListener` によりダメージ倍率（1.2倍）が適用されること。3本の矢がそれぞれ独立して命中判定・ダメージ計算されることを確認する。
- **`explosive_arrow`**（`ExplosiveArrowExecutor`）: メタデータ `orelia_skill_explosive_arrow` = `double[]{amount, radius}` が付与され、`ExplosiveArrowHitListener`（`ProjectileHitEvent`）で着弾時に半径3.0のAoEダメージが発生すること。ブロックへの着弾でも爆発すること、プレイヤー未命中でも起爆自体はすることを確認する。

## 異常系・エッジケース

- 弓のスキルをソケット・習得した状態で、剣を持ち替えて発動しようとすると `WRONG_WEAPON` になること。
- スキルスロット数（`skill-slot-count`）を超えてソケットしようとした場合の挙動（拒否される、または上書きされる）を確認し、`items.yml` の値（1〜2個）どおりに制限されているか確認する。
- 同じ武器を複数本（スタック）所持している場合、ソケット状態はスタック内の代表アイテムだけに反映され、他の個体には反映されないこと（PDCはアイテム個体ごとの状態のため）。
