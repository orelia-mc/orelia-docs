# Job（職業）の動作確認

対象: `rpg.job`（[Job モジュール仕様](../core/job.md)）。職業は経験値を持たない静的な状態で、武器種の使用制限とパッシブなステータス補正だけを与えます。

## 前提条件

- `orelia-core` 単体には**職業変更コマンドが無い**ことに注意してください。職業変更は `orelia-world` の `JOB_CHANGE` 型NPC（`guiApi.openJobChange` → `JobGuiScreen`）を右クリックして行います（[NPC モジュール](../world/npc.md)）。
    - `orelia-world` が未導入、またはNPCがまだ設置されていない場合は、テスト用に `npc.yml` へ `type: JOB_CHANGE` のNPCを1体追加し、`/rpgworldadmin reload` 後にワールドへスポーンさせて使ってください。
- 確認には各職業に対応するテスト武器が便利です（`items.yml` 標準定義）：

    | 職業 | 許可武器 | テスト用武器 | 必要レベル |
    |---|---|---|---|
    | SWORDSMAN（剣士） | SWORD | `wooden_training_sword`（Lv1） | 1 |
    | SPEARMAN（槍使い） | SPEAR | `hunters_spear`（Lv5） | 5 |
    | WARRIOR（戦士） | AXE, SWORD | `berserkers_axe`（Lv8） | 8 |
    | ARCHER（弓使い） | BOW | `hawkeye_bow`（Lv6） | 6 |

    レベルが足りない場合は `WeaponRequirementService.meetsRequirements` により装備・使用が拒否されるはずです（下記「異常系」参照）。

## 1. 職業一覧・現在の職業表示

```
/ol job list
```

期待結果: `SWORDSMAN`, `SPEARMAN`, `WARRIOR`, `ARCHER` の4種が（`jobs.yml` の `display-name` 付きで）表示される。

```
/ol job
```

- 未就業（`currentJob == null`）の場合: 「まだ選択されていません。NPCを訪ねてください」的なメッセージ。
- 就業済みの場合: 現在の職業名が表示される。

## 2. 職業変更（NPC経由）

1. `JOB_CHANGE` NPCに右クリックで話しかける → `JobGuiScreen`（27スロット、タイトル「職業変更」）が開く。
2. 現職業のアイコンが `GOLDEN_HELMET`、他の職業が `LEATHER_HELMET` で表示されていることを確認。
3. 未就業の状態から `SWORDSMAN` をクリック → GUIが閉じ、`/ol job` で `SWORDSMAN` になっていることを確認。
4. 続けて `WARRIOR` をクリックして再変更 → `/ol job` の結果が `WARRIOR` に切り替わることを確認（**要件チェックなしで即座に切り替わる**のが仕様。装備中の武器が新職業で使えなくなっても変更自体は成功する）。

## 3. 武器種制限の確認

各職業ごとに、許可されていない武器種を使おうとして拒否されることを確認します。

1. `SWORDSMAN` の状態で `/ol item give <自分> hunters_spear 1`（SPEARはSWORDSMANの許可武器外）で入手。
2. 装備・攻撃を試み、`JobService.canUseWeaponType` により使用不可となること（通常攻撃が通らない、またはペナルティが掛かる）を確認。
3. `WARRIOR` に変更し、同じ手順で `wooden_training_sword`（SWORD）と `berserkers_axe`（AXE）の両方が使用可能なことを確認（`WARRIOR` は `allowed-weapons: [AXE, SWORD]` の唯一2種持ち職業）。

## 4. パッシブ補正の確認

`jobs.yml` の `passive-bonus` は `StatusService.setEquipmentContribution(uuid, "job", passiveBonus)` として反映されます（ソースキー `"job"`）。

1. 職業変更前に `/ol status` でATK（と該当ステータス）の値をメモする。
2. 職業を変更する。
3. 再度 `/ol status` を開き、以下のとおり変化していることを確認する：

    | 職業 | 期待される補正 |
    |---|---|
    | SWORDSMAN | ATK +2 |
    | SPEARMAN | ATK +1 |
    | WARRIOR | ATK +3, DEF +1 |
    | ARCHER | 補正なし（`passive-bonus` 未設定） |

4. 別の職業に切り替えたとき、前の職業の補正が消え、新しい職業の補正だけが乗っていること（同じソースキー `"job"` で上書きされるため、加算されて残らないこと）を確認する。

## 5. 技（スキル）の動作確認

職業ごとの武器種制限は、そのままその職業で使える技（武器スキル）の種類を決めます。実際の技の発動・クールダウン・SP消費・レベルアップの確認は [Skill の確認手順](skill.md) にまとめています。職業別の対応は以下のとおりです。

| 職業 | 使える武器種 | 対応する技（`skills.yml`） |
|---|---|---|
| SWORDSMAN | SWORD | 斬撃 `slash` / 回転斬り `spin_slash` / 居合 `iaijutsu` / クロススラッシュ `cross_slash` |
| SPEARMAN | SPEAR | 突進 `charge_thrust` / スパイラルランス `spiral_lance` / ジャンプ突き `jump_thrust` |
| WARRIOR | AXE, SWORD | バーサーク `berserk` / アースクラッシュ `earth_crush` / 大振り `big_swing`（AXE側）+ SWORD側の4技 |
| ARCHER | BOW | パワーショット `power_shot` / マルチショット `multi_shot` / 爆裂矢 `explosive_arrow` |

職業を切り替えるだけでは技はソケットも習得もされません。`SkillGuiScreen`（技のソケット・習得を行う画面）でのソケット・習得操作が別途必要です。**現時点のコードベースでは `SkillGuiScreen` を開くプレイヤー向けの導線（コマンドやNPC）が存在しない**既知の未接続ギャップがあります。詳細と検証時の回避策は [Skill の確認手順](skill.md) を参照してください。

## 異常系・エッジケース

- **必須レベル未満の武器を渡す**: `/ol item give` 自体は成功しますが（管理者コマンドに要件チェックは無い）、実際に装備・攻撃しようとすると `WeaponRequirementService.meetsRequirements` の必須レベルチェックで弾かれることを確認します（例: レベル1の状態で `berserkers_axe`（要Lv8）を持たせる）。
- **必須職業と異なる職業で武器を使う**: `flame_longsword`（`required-job: SWORDSMAN`）を `ARCHER` の状態で使用しようとして拒否されることを確認。
- **`changeJob` に要件ゲートが無いこと**: 装備を何も持たない・レベルが足りない状態でも職業変更自体は成功する仕様です（`JobService.changeJob` は要件チェックをしません）。これはバグではなく仕様なので、「職業だけ変わって武器が使えない」状態になっても正常です。
