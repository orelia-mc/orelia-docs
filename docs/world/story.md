# Story モジュール（`rpg.world.story`）

章/シナリオ進行と、フラグ（多くはクエスト報酬やダイアログ選択によって設定される）による複数エンディング解禁を管理する、全体的な物語進行トラッカーです。

## ドメインモデル

```java
class StoryChapter {
    String id;
    int order;
    String title;
    List<String> description;
    List<String> requiredFlags;
    String unlockMessage;
}

class StoryEnding {
    String id, title;
    List<String> message;
    List<String> requiredFlags;
}

class PlayerStoryComponent implements PlayerDataComponent {
    String currentChapterId;
    Set<String> completedChapterIds;
    Set<String> flags;
    Set<String> unlockedEndingIds;

    boolean hasFlag(String flag);
    void setFlag(String flag);
    void completeChapter(String chapterId);
    void unlockEnding(String endingId);
}
```

## `story.yml` の実際のキー

```yaml
chapters:
  chapter_2:
    order: 2
    title: "第二章 - ゴブリンの脅威"
    required-flags: ["cleared_goblin_supplies"]
    unlock-message: "&6&l第二章「ゴブリンの脅威」が始まりました。"
endings:
  true_ending:
    title: "真のエンディング"
    message: ["&d&l祝福のエンディングを迎えました！"]
    required-flags: ["defeated_goblin_king"]
```

## サービス

- **`StoryRepository`** — `getChaptersInOrder()`（`order` でソート済み）, `getEndings()`
- **`PlayerStoryRepository`**（`SchemaOwner`） — テーブル `story_progress`, `story_completed_chapter`, `story_flag`, `story_ending`
- **`StoryManager`**（`PlayerDataComponentLoader`）
- **`StoryProgressService`**
    - `setFlag`, `hasFlag`
    - `advanceChapter(Player, completedChapterId)` — 完了マーク後、未完了の章のうち必要フラグを全て満たす最も早い章を探し、現在の章として設定、解禁メッセージを送信、その後 `checkEndings` を実行
    - `getCurrentChapter(UUID)`

## 消費する orelia-core API

`rpg.api.*` は使用しません。汎用の `DatabaseManager` のみ利用します。
