# Job（職業）の動作確認

対象: `rpg.job`（[Job モジュール仕様](../core/job.md)）。職業は経験値を持たない静的な状態で、武器種の使用制限とパッシブなステータス補正だけを与えます。

!!! info "職業ロースターが3種に変更されています"
    `SPEARMAN`（槍使い）は廃止され、現在の職業は `FENCER`（フェンサー）/`WARRIOR`（ウォーリアー）/`ARCHER`（アーチャー）の3種のみです。`WARRIOR` も以前の「AXE, SWORD」の2種持ちから **AXE専用**に変更されています。SPEAR武器種自体（`hunters_spear`）はまだ `items.yml`/`WeaponType` に残っていますが、**この3職業のうちどれも `allowed-weapons` にSPEARを含んでいない**ため、現状SPEAR武器はどの職業でも武器ダメージを出せません（`items.yml` の該当コメント参照）。これはバグではなく現時点のconfig内容であり、テスト時に「槍だけ攻撃が通らない」という結果が出ても正常です。

## 前提条件

- `orelia-core` 単体には**職業変更コマンドが無い**ことに注意してください。職業変更は `orelia-world` の `JOB_CHANGE` 型NPC（`guiApi.openJobChange` → `JobGuiScreen`）を右クリックして行います（[NPC モジュール](../world/npc.md)）。
- **`orelia-world` 導入済みでも、このNPC（`npc.yml` の `job_master`）はサーバー起動時に自動配置されません。** `NpcSpawnSyncService` は `JOB_CHANGE` タイプだけを起動時同期の対象から除外する仕様のため、初回は管理者が手動で以下を実行して配置する必要があります。

    ```
    /oladmin spawnnpc job_master
    ```

    実行した場所に「職業指南役ヴィクター」が湧きます。他のNPC（ショップ・鍛冶屋・倉庫番・クエスト受付・ギルド受付）は `npc.yml` の座標に起動時自動配置されるため、この手順が必要なのは `JOB_CHANGE` タイプだけです。
- 確認には各職業に対応するテスト武器が便利です（`items.yml` 標準定義）：

    | 職業 | 表示名 | 許可武器 | テスト用武器 | 必要レベル |
    |---|---|---|---|---|
    | FENCER | フェンサー | SWORD | `wooden_training_sword`（Lv1） | 1 |
    | WARRIOR | ウォーリアー | AXE | `berserkers_axe`（Lv8） | 8 |
    | ARCHER | アーチャー | BOW | `hawkeye_bow`（Lv6） | 6 |

    レベルが足りない場合は `WeaponRequirementService.meetsRequirements` により装備・使用が拒否されるはずです（下記「異常系」参照）。`hunters_spear`（SPEAR, Lv5）は上記の理由によりどの職業でも通常攻撃が通らないため、テスト対象から除外しています。

## 1. 職業一覧・現在の職業表示

```
/ol job list
```

期待結果: `FENCER`, `WARRIOR`, `ARCHER` の3種が（`jobs.yml` の `display-name`＝カタカナ表記付きで）表示される。`SWORDSMAN`/`SPEARMAN` は表示されない。

```
/ol job
```

- 未就業（`currentJob == null`）の場合: 「まだ選択されていません。NPCを訪ねてください」的なメッセージ。
- 就業済みの場合: 現在の職業名（カタカナ表示名）が表示される。

## 2. 職業変更（NPC経由）

1. `job_master`（`JOB_CHANGE`）NPCに右クリックで話しかける → `JobGuiScreen`（27スロット、タイトル「職業変更」）が開く。
2. 現職業のアイコンが `GOLDEN_HELMET`、他の職業が `LEATHER_HELMET` で表示されていることを確認。
3. 未就業の状態から `FENCER` をクリック → GUIが閉じ、`/ol job` で `フェンサー` になっていることを確認。
4. 続けて `WARRIOR` をクリックして再変更 → `/ol job` の結果が `ウォーリアー` に切り替わることを確認（**要件チェックなしで即座に切り替わる**のが仕様。装備中の武器が新職業で使えなくなっても変更自体は成功する）。

## 3. 武器種制限の確認

各職業ごとに、許可されていない武器種を使おうとして拒否されることを確認します。

1. `FENCER` の状態で `/ol item give <自分> berserkers_axe 1`（AXEはFENCERの許可武器外）で入手。
2. 装備・攻撃を試み、`JobService.canUseWeaponType` により使用不可となること（通常攻撃が通らない、またはペナルティが掛かる）を確認。
3. `WARRIOR` に変更し、`berserkers_axe`（AXE）が使用可能、`wooden_training_sword`（SWORD）は使用不可になることを確認する（`WARRIOR` は AXE専用に変更されているため、以前と違いSWORDは使えない）。

## 4. パッシブ補正の確認

`jobs.yml` の `passive-bonus` は `StatusService.setEquipmentContribution(uuid, "job", passiveBonus)` として反映されます（ソースキー `"job"`）。

1. 職業変更前に `/ol status` でATK（と該当ステータス）の値をメモする。
2. 職業を変更する。
3. 再度 `/ol status` を開き、以下のとおり変化していることを確認する：

    | 職業 | 期待される補正 |
    |---|---|
    | FENCER | ATK +2 |
    | WARRIOR | ATK +3, DEF +1 |
    | ARCHER | 補正なし（`passive-bonus` 未設定） |

4. 別の職業に切り替えたとき、前の職業の補正が消え、新しい職業の補正だけが乗っていること（同じソースキー `"job"` で上書きされるため、加算されて残らないこと）を確認する。

## 5. 技（スキル）の動作確認

職業ごとの武器種制限は、そのままその職業で使える技（武器スキル）の種類を決めます。実際の技の発動・クールダウン・SP消費・レベルアップの確認は [Skill の確認手順](skill.md) にまとめています。職業別の対応は以下のとおりです。

| 職業 | 使える武器種 | 対応する技（`skills.yml`） |
|---|---|---|
| FENCER | SWORD | 斬撃 `slash` / 回転斬り `spin_slash` / 居合 `iaijutsu` / クロススラッシュ `cross_slash` |
| WARRIOR | AXE | バーサーク `berserk` / アースクラッシュ `earth_crush` / 大振り `big_swing` |
| ARCHER | BOW | パワーショット `power_shot` / マルチショット `multi_shot` / 爆裂矢 `explosive_arrow` |

職業を切り替えるだけでは技はソケットも習得もされません。技のソケット・習得・発動自体の確認手順は [Skill の確認手順](skill.md) にまとめています（`orelia-world` 導入時は、常時携帯する「プレイヤー情報」ネザースター経由でも技画面に到達できます）。

## 6. Gathering（採取/農業）レベルアップ時の職業名表示の確認

職業とは直接関係ありませんが、`GatheringLevelService` はレベルアップ通知メッセージに現在の職業表示名を使います（未就業なら「無職」）。[Gathering の確認手順](gathering.md) と合わせて、職業変更後にこのメッセージの職業名が正しく切り替わることを確認しておくと良いです。

## 異常系・エッジケース

- **必須レベル未満の武器を渡す**: `/ol item give` 自体は成功しますが（管理者コマンドに要件チェックは無い）、実際に装備・攻撃しようとすると `WeaponRequirementService.meetsRequirements` の必須レベルチェックで弾かれることを確認します（例: レベル1の状態で `berserkers_axe`（要Lv8）を持たせる）。
- **必須職業と異なる職業で武器を使う**: `flame_longsword`（`required-job: FENCER`）を `ARCHER` の状態で使用しようとして拒否されることを確認。
- **`changeJob` に要件ゲートが無いこと**: 装備を何も持たない・レベルが足りない状態でも職業変更自体は成功する仕様です（`JobService.changeJob` は要件チェックをしません）。これはバグではなく仕様なので、「職業だけ変わって武器が使えない」状態になっても正常です。
- **`hunters_spear` が誰にも使えないこと**: 上述のとおり既知の内容ギャップです。バグ報告する場合は「SPEARを許可する職業が無い」ことがconfig起因か仕様かをまず確認してください。
