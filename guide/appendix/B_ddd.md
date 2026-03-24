# Appendix B: DDD（ドメイン駆動設計）

## DDDとは

DDD（Domain-Driven Design）は、ソフトウェアの設計をビジネスの「ドメイン（領域）」に合わせて行う設計手法である。
技術的な都合ではなく、**業務の言葉・構造をそのままコードに反映する**ことが目的。

AIとの協調開発においては、DDDの概念を「AIへ渡す文脈の単位を設計するための道具」として活用する。

DDDには大きく2つの側面がある。

```
戦略的設計（Strategic Design）
  → ドメインをどう分割するか。境界付けられたコンテキスト、コンテキストマップ
  → 「何を作るか」の構造を決める

戦術的設計（Tactical Design）
  → 分割されたドメイン内部をどうモデリングするか。エンティティ、値オブジェクト、集約
  → 「どう作るか」の構造を決める
```

**最も重要なのは戦略的設計（ドメインの発見と分割）であり、エンティティや集約を書くこと自体ではない。**
戦術的設計の成果物は、発見プロセスの結果として自然に生まれるものである。

---

## DDDの発見プロセス: イベントストーミング

### なぜ発見が先か

DDDの成果物（ユビキタス言語、ドメインモデル等）をいきなり書こうとすると、以下の問題が起きやすい。

- ドメインの全体像が見えないまま、断片的なモデルができる
- 重要なドメインイベントやポリシーが見落とされる
- 集約の境界が技術的な都合で決まり、業務の構造と乖離する

イベントストーミング（`appendix/C_event_storming.md` で詳説）を先にやることで、
**ドメインの全体像が可視化され、モデルの境界が業務に沿って決まる**。

### イベントストーミングからDDD成果物への流れ

```
イベントストーミング          DDD成果物
───────────────            ──────────
ドメインイベント一覧     →   commands_and_events.md
コマンド一覧             →   commands_and_events.md
ポリシー                 →   commands_and_events.md
集約のグルーピング       →   domain_model.md（集約の境界）
業務用語の統一           →   ubiquitous_language.md
問題・疑問（未解決）      →   TODOコメント or 次の発見セッションへ
```

### テキストベースでの発見

物理的な付箋が使えない場合、Markdownでイベントストーミングを実施できる。

```markdown
# イベントストーミング: 注文管理システム

## ドメインイベント（時系列）

1. OrderPlaced（注文が置かれた）
2. PaymentReceived（支払いが受領された）
3. OrderConfirmed（注文が確定した）
4. KitchenNotified（キッチンに通知された）
5. DishPrepared（料理が調理された）
6. OrderServed（注文が提供された）
7. PaymentCompleted（支払いが完了した）

## コマンドとアクター

| コマンド | アクター | トリガーとなるイベント |
|---------|---------|---------------------|
| PlaceOrder | ウェイター | — （開始点） |
| ReceivePayment | システム | OrderPlaced |
| ConfirmOrder | システム | PaymentReceived |
| NotifyKitchen | ポリシー | OrderConfirmed |
| PrepareDish | キッチン | KitchenNotified |
| ServeOrder | ウェイター | DishPrepared |

## ポリシー

| イベント | ポリシー | 実行されるコマンド |
|---------|---------|------------------|
| OrderConfirmed | 注文確定時にキッチンへ自動通知 | NotifyKitchen |
| PaymentReceived | 支払い受領時に注文を自動確定 | ConfirmOrder |

## 集約の候補

- **Order集約**: PlaceOrder, OrderPlaced, OrderConfirmed, OrderServed
- **Payment集約**: ReceivePayment, PaymentReceived, PaymentCompleted
- **Kitchen集約**: NotifyKitchen, KitchenNotified, DishPrepared

## 疑問

- [ ] 支払い前に注文をキャンセルできるか？
- [ ] 部分的な支払いは許容するか？
- [ ] 複数テーブルをまとめて支払えるか？
```

このテキストが、DDD成果物の**設計図**になる。

### AIとのイベントストーミング

AIを壁打ち相手としてイベントストーミングを進められる。

```
以下のシステムについてイベントストーミングを一緒にやりたいです。

システム: レストランの注文管理

ドメインイベントを時系列で洗い出してください。
その後、各イベントのコマンドとアクターを特定してください。
特に『このイベントの後に自動的に起きることは何か？』（ポリシー）を重視してください
```

AIはイベント・コマンド・ポリシーの候補を列挙する。
**人間はドメインの妥当性と業務ルールの正しさを判断する。**

### 雑多なメモからのイベントストーミング成果生成

壁打ちや議論で出たメモから、イベントストーミングの成果物に整理できる。

```
以下はシステムについての壁打ちメモ・議論ログです。
 これをイベントストーミングの成果物として整理してください。

 フォーマット:
 1. ドメインイベント（時系列順、過去形で命名）
 2. コマンドとアクター（各イベントのトリガー）
 3. ポリシー（イベント発生時に自動実行されるルール）
 4. 集約の候補（関連するコマンド・イベントのグルーピング）
 5. 疑問（メモの中で結論が出ていないもの）

 メモに明示されていないが業務上ありそうなイベント（暗黙のイベント）も
 候補として提示してください。

 ---
 [壁打ちメモ・議論ログを貼る]
```

整理されたイベントストーミング成果から、続けてDDD成果物の生成を依頼できる。

```
上のイベントストーミングの成果を、以下の3ファイルに変換してください。

 1. ubiquitous_language.md — イベント・コマンド・集約で使った用語を定義一覧に
 2. commands_and_events.md — コマンド・イベント・ポリシーを表形式で整理
 3. domain_model.md — 集約の境界・エンティティ・値オブジェクトを定義

 疑問の項目は各ファイル内に # TODO コメントとして残してください。
```

> **ポイント:** 2段階に分けることで、「メモ → イベントストーミング成果」の段階で
> 人間がイベントの妥当性やポリシーの正しさを確認でき、
> いきなりドメインモデルが出てくるよりもレビューしやすくなる。

---

## 発見からDDD成果物への変換

### 変換の流れ

イベントストーミングの成果物は、ほぼ機械的にDDD成果物に変換できる。

```
イベントストーミング成果          DDD成果物ファイル
───────────────────            ────────────────
業務用語の合意              →   ubiquitous_language.md
コマンド・イベント・ポリシー  →   commands_and_events.md
集約の境界・エンティティ      →   domain_model.md
未解決の疑問                →   TODOとして各ファイルに記録
```

### AIによるDDD成果物生成支援

イベントストーミングのテキストを渡せば、AIがDDD成果物に変換できる。

```
以下のイベントストーミングの成果を、DDD成果物に変換してください。

1. ubiquitous_language.md — イベントストーミングで使った用語を定義一覧にしてください
2. commands_and_events.md — コマンド・イベント・ポリシーを整理してください
3. domain_model.md — 集約の境界とエンティティ・値オブジェクトを定義してください

[イベントストーミングのテキストを貼る]
```

> **ポイント:** イベントストーミングなしにいきなり「ドメインモデルを書いてください」と
> AIに頼むと、技術的な構造のモデルが生成されやすい。
> イベントストーミングを先にやることで、**業務の構造に沿ったモデルが生まれる**。

---

## DDDの主要概念

### ユビキタス言語

プロジェクト内で使う言葉を統一したもの。
開発者・ユーザー・AIの全員が同じ言葉を使うことで、認識のズレを防ぐ。

```
例:
  「注文」  = テーブルに紐づく、1回の食事における料理の注文のまとまり
  「配膳」  = 調理が完了した料理をテーブルに届けること
```

### エンティティ

一意に識別できるオブジェクト。状態が変化しても同一として扱う。

```
例: Order（注文）
  → IDで識別される
  → 状態（注文済み・調理中・提供済み・会計済み）が変化しても同じ注文
```

### 値オブジェクト

識別子を持たず、値そのものが意味を持つオブジェクト。不変。

```
例: TableNumber（テーブル番号）
  → "A-3" という値そのものが意味を持つ
  → 変更したら別の値オブジェクトになる
```

### 集約

関連するエンティティ・値オブジェクトをまとめた単位。集約ルートを通じてのみアクセスする。

```
例: Order（集約ルート）
  ├── TableNumber
  ├── []OrderItem
  └── OrderStatus
```

### ドメインイベント

ドメイン内で起きた出来事。過去形で命名する。

```
例: OrderPlaced, DishPrepared, PaymentCompleted
```

### コマンド

ユーザーの意図・システムへの指示。ドメインイベントを引き起こす。

```
例: PlaceOrder, PrepareDish, ProcessPayment
```

### 境界付けられたコンテキスト

ドメインモデルが有効な範囲。大規模システムではコンテキストを分割して管理する。

```
例:
  注文コンテキスト  → Order, OrderItem, TableNumber
  キッチンコンテキスト → Dish, CookingQueue, Recipe
  会計コンテキスト  → Payment, Receipt, Bill
```

---

## DDDで作成するファイル一覧

```
documents/requirements/domain/
├── ubiquitous_language.md  # ユビキタス言語
├── commands_and_events.md  # コマンド・イベント・ポリシー
└── domain_model.md         # エンティティ・値オブジェクト・集約
```

---

## ubiquitous_language.md の作り方

プロジェクトで使う言葉の定義一覧。
開発が進むにつれて育てていくドキュメント。

```markdown
# ユビキタス言語

## ○○関連

| 用語 | 定義 |
|------|------|
| **用語A** | 定義の説明。曖昧さが残らないように具体的に書く |
| **用語B** | 定義の説明 |

## △△関連

| 用語 | 定義 |
|------|------|
| **用語C** | 定義の説明 |
```

### 書くときのポイント

- 開発者の言葉ではなくユーザーの言葉で書く
- 同じ意味で複数の言葉を使わない（「ジョブ」と「タスク」を混在させない）
- 曖昧な定義は書かない（「適切な処理をするもの」は不可）

---

## commands_and_events.md の作り方

コマンド（ユーザーの意図）とドメインイベント（起きた出来事）の一覧。
イベントストーミングの成果をまとめる場所。

```markdown
# コマンドとドメインイベント

## コマンド一覧

| コマンド | トリガー | 説明 |
|---------|---------|------|
| **CommandA** | ○○操作 | 何をするコマンドか |

## ドメインイベント一覧

### CommandA フロー

CommandA
  └─→ EventX    説明
  └─→ EventY    説明

## ポリシー（Policy）

イベントが発生したときに自動的に実行されるルール。

| イベント | ポリシー |
|---------|---------|
| EventX | 何を自動実行するか |
```

### 書くときのポイント

- コマンドは命令形（PlaceOrder）
- イベントは過去形（OrderPlaced）
- ポリシーは「〜したとき、〜する」の形で書く

---

## domain_model.md の作り方

エンティティ・値オブジェクト・集約の定義。コードの設計図になる。

```markdown
# ドメインモデル

## エンティティ・値オブジェクト

### EntityA（エンティティ）
説明を1行で書く。

​```
EntityA
  - id: EntityAId       # 一意識別子
  - field1: Type        # フィールドの説明
  - method1() Type      # メソッドの説明
​```

### ValueObjectB（値オブジェクト）
説明を1行で書く。

​```
ValueObjectB
  - value: string
  - validate() bool
​```

## 状態遷移

### EntityAのステータス

​```
StateA → StateB（条件）→ StateC
​```

## 集約の境界

​```
AggregateRoot（集約ルート）
  ├── EntityA
  ├── ValueObjectB
  └── []EntityC
​```
```

### 書くときのポイント

- 集約ルートを明確にする
- 状態遷移は図（テキスト図）で示す
- 実装言語に依存しない書き方にする（GoでもPythonでも読める）

---

## コーディングAIへの渡し方

DDD成果物は、そのままAIへの実装指示として使える。

```
documents/requirements/domain/ のすべてのDDD成果物を読んだ上で、
 domain_model.md に定義された集約を実装してください。
 ubiquitous_language.md の用語をそのままクラス名・変数名に使ってください。
```

DDD成果物が具体的であるほど、AIの実装精度が上がる。

---

## 参考文献

### DDDの原典

- [Eric Evans, "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003)](https://www.domainlanguage.com/ddd/) — DDDの提唱者による原典。ユビキタス言語、境界付けられたコンテキスト、集約等の概念を体系化
- [Eric Evans, "DDD Reference" (2015)](https://www.domainlanguage.com/ddd/reference/) — DDDのパターンカタログ。無料で入手可能な公式リファレンス

### 実践ガイド

- [Vaughn Vernon, "Implementing Domain-Driven Design" (2013)](https://www.informit.com/store/implementing-domain-driven-design-9780321834577) — 通称「IDDD本」。DDDを実際のプロジェクトに適用する方法を詳説
- [Vaughn Vernon, "Domain-Driven Design Distilled" (2016)](https://www.informit.com/store/domain-driven-design-distilled-9780134434421) — DDDのエッセンスを凝縮した入門書。戦略的設計と戦術的設計の概要

### 関数型・軽量アプローチ

- [Scott Wlaschin, "Domain Modeling Made Functional" (2018)](https://pragprog.com/titles/swdddf/domain-modeling-made-functional/) — 関数型プログラミングでDDDを実践するアプローチ。型でドメインを表現する手法

### 発見プロセス

- [Alberto Brandolini, "Introducing EventStorming" (2021)](https://www.eventstorming.com/book/) — イベントストーミングの提唱者による解説。DDDの発見プロセスとして最も広く使われている手法
- [Nick Tune & Eduardo da Silva, "Architecture Modernization" (2024)](https://www.manning.com/books/architecture-modernization) — DDDとイベントストーミングを活用したアーキテクチャ近代化の実践ガイド

### コミュニティ・リソース

- [DDD Community](https://www.dddcommunity.org/) — DDDコミュニティの公式サイト。パターン・事例・議論のハブ
- [Virtual DDD](https://virtualddd.com/) — DDDのオンライン勉強会・セッション。実践者による知見共有
