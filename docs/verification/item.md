# Item（武器）の動作確認

対象: `rpg.item`（[Item モジュール仕様](../core/item.md)）。`items.yml` の武器テンプレートから `ItemStack` を生成し、PDCで識別、職業/レベル要件を検証します。

## 前提条件

OPで検証します。`items.yml` 標準定義のサンプル武器を使います。

| ID | 表示名 | 種別 | Lv | レア度 | 必須職業 |
|---|---|---|---|---|---|
| `wooden_training_sword` | 見習いの剣 | SWORD | 1 | COMMON | FENCER |
| `flame_longsword` | 紅蓮の長剣 | SWORD | 10 | RARE | FENCER |
| `hunters_spear` | 狩人の槍 | SPEAR | 5 | UNCOMMON | （なし＝職業不問） |
| `berserkers_axe` | 狂戦士の斧 | AXE | 8 | RARE | WARRIOR |
| `hawkeye_bow` | 鷹の眼の弓 | BOW | 6 | UNCOMMON | ARCHER |

!!! note "`hunters_spear` は現状どの職業でも武器ダメージが出ません"
    `required-job` は未設定（職業不問）ですが、現在の職業ロースター（FENCER/WARRIOR/ARCHER）の `allowed-weapons` にSPEARを含むものが1つも無いため、`JobService.canUseWeaponType` が常にfalseを返します。装備自体は誰でもできますが、通常攻撃・武器スキルともに弾かれるのが現状の仕様です。詳細は [Job の確認手順](job.md) を参照してください。

## 1. 武器生成コマンド

```
/ol item give <自分の名前> wooden_training_sword 1
```

期待結果:

- インベントリに「見習いの剣」が1本追加される。
- アイテムのLore/名前に `items.yml` の `description`・レア度カラー（COMMONの色）が反映されている。
- 個数指定を省略した場合は `1` 本になる（デフォルト値）。
- 存在しないID（例: `/ol item give <自分> nonexistent_sword`）を渡すとエラーメッセージが返り、アイテムは生成されない。
- 実行者以外のオンラインプレイヤー名を指定して正しく相手に渡ること。

!!! note "権限チェックなし"
    `ItemCommand` にはコード上明示的な権限チェックがありません。OP以外に実行させたくない場合は運用（パーミッションプラグイン等）側で制御してください。

## 2. 識別（PDC）の確認

1. 生成した武器を一度インベントリから取り出し、別のスロットに移動してもPDC情報が保持されている（アイテムスタックのNBT/PDCに `weapon_id` タグが残っている）ことを確認する。`EquipmentGuiScreen`（装備画面）は現時点で `orelia-core`/`orelia-world` のどちらからも開く導線が無い（[GUI の確認手順](gui.md)参照）ため、[Skill の確認手順](skill.md)にある `SkillGuiScreen` 経由で武器が正しく識別されることで代わりに確認すると良い。
2. バニラの通常アイテム（例: 素手のダイヤの剣）を持った状態で `SkillGuiScreen` を開くと BARRIER（識別不可）表示になり、Orelia武器では正しくスキル一覧が出ることを確認する（`WeaponIdentityService.isOreliaWeapon` の境界確認）。

## 3. 職業/レベル要件の確認

`WeaponRequirementService.meetsRequirements` の確認です。

1. 未就業（職業なし）の状態で `wooden_training_sword` を装備・攻撃しようとし、拒否されることを確認。
2. `FENCER` かつレベル1未満はあり得ないので、`flame_longsword`（要Lv10）をレベル1のFENCERに持たせて使用を試み、要件不足で拒否されることを確認。
3. `FENCER` でレベル10到達後、同じ武器が使用可能になることを確認。
4. 職業が一致しない場合（例: `ARCHER` が `flame_longsword` を使用）に拒否されることを確認。

## 4. 強化（enhancement）の確認

`WeaponIdentityService.enhance` はPDC `enhancement_level` をインクリメントし、`enhancementMultiplier = 1.0 + level * 0.1` をダメージ計算に反映します。

1. `orelia-world` の `ENHANCEMENT` 型NPC（`blacksmith` など）に武器を持って話しかけ、必要な `Vault` 経済コストを払って強化を実行する（[NPC モジュール](../world/npc.md)参照）。
2. 強化後、武器のLore等に強化レベルが反映されているか、または攻撃時のダメージが強化前より約10%/レベル増えているかを確認する（[Status のダメージ計算式](status.md)と合わせて確認）。
3. `orelia-core` 単体（`orelia-world` 未導入）では強化を試す標準UIが無いため、この項目はスキップして構いません。

## 5. 売却価格・レアリティ・属性表示の確認

各サンプル武器を生成し、Lore/アイテム名に以下が反映されていることを目視確認します。

- レア度に応じたカラー（COMMON/UNCOMMON/RARE/EPIC/LEGENDARY で異なる `ChatColor`）。
- `element`（例: `flame_longsword` は FIRE、`hawkeye_bow` は WIND）。
- `custom-model-data` が設定されていること（リソースパック未適用でも、アイテムのNBTとしては入っている）。

売却は `orelia-world` のショップNPC（`WEAPON_SHOP` 等）を通して行い、`sell-price` どおりの金額が `EconomyService` に反映されることを [Economy の確認手順](economy.md) と合わせて確認してください。

## 異常系・エッジケース

- `/ol item give` に個数として負数や0を渡した場合の挙動（想定外の値なので、エラー・無視・そのまま処理される、のいずれかを確認して記録しておく）。
- 同じ武器を複数所持した状態でスキルスロット数（`skill-slot-count`）がスタック単位ではなく**アイテム個体**単位で管理されているか（ソケット済みスキルはPDC文字列としてアイテムに直接書き込まれるため、スタック不可になるはず）を [Skill の確認手順](skill.md) と合わせて確認する。
