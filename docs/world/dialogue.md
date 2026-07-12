# Dialogue モジュール（`rpg.world.dialogue`）

分岐会話木とプレイヤーごとの永続フラグ。他モジュール（story, npc）が参照するナラティブ/フラグレイヤーです。

## ドメインモデル

```java
class DialogueTree {
    String id;
    List<DialogueEntryPoint> entryPoints;
    Map<String, DialogueNode> nodes;
}

class DialogueEntryPoint {
    String requiredFlag, startNodeId;
    // config順で最初に、プレイヤーがそのフラグを持つ（またはフラグが空文字の）エントリーポイントが採用される
}

class DialogueNode {
    String id;
    List<String> lines;
    List<DialogueChoice> choices;
    String nextNodeId; // 選択肢無しの場合の直線的な遷移先
    String setFlag;    // このノード表示時に設定されるフラグ
}

class DialogueChoice {
    String text, nextNodeId, setFlag;
}

class PlayerDialogueComponent implements PlayerDataComponent {
    Set<String> flags;
    boolean hasFlag(String flag);
    void setFlag(String flag);
}
```

## `dialogues.yml` の実際のキー

```yaml
dialogues:
  quest_receptionist_intro:
    entry-points:
      returning:  { required-flag: "met_receptionist", start-node: welcome_back }
      first_time: { required-flag: "", start-node: first_meeting }
    nodes:
      first_meeting:
        lines: ["あら、見ない顔ね。この町は初めて？"]
        set-flag: "met_receptionist"
        choices:
          yes: { text: "はい、初めてです", next-node: explain_quests }
```

## サービス

- **`DialogueRepository`** — `load`/`findById`
- **`PlayerDialogueRepository`**（`SchemaOwner`） — テーブル `dialogue_flag`。`loadOrCreate`/`save`
- **`DialogueManager`**（`PlayerDataComponentLoader<PlayerDialogueComponent>`）
- **`DialogueSessionService`** — 実行時のみの `Map<UUID, ActiveSession(treeId, nodeId)>`
    - `start(Player, treeId)`
    - `choose(Player, int choiceIndex)`
    - `showNode`（private） — セリフ表示、選択肢をクリック可能なAdventure `Component`（`ClickEvent.runCommand("/dialoguechoice " + i)`）として描画

## コマンド

`DialogueChoiceCommand` → `/dialoguechoice <index>`（内部用。チャット上のクリック可能な行から呼ばれる）

## 消費する orelia-core API

`rpg.api.*` のゲームプレイAPIは使用しません。汎用インフラの `DatabaseManager` のみ利用します。
