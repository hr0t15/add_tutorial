# Appendix C: イベントストーミング

## イベントストーミングとは

イベントストーミングは、ドメインイベントを起点にシステムの全体像を発見するワークショップ手法である。
Alberto Brandoliniが考案した。

「何が起きるか」を時系列で並べることで、システムの振る舞い・構造・問題点を短時間で可視化できる。

AIとの協調開発では、**振る舞い定義（BDD）の次のステップ**として活用する。
シナリオが「何をするか」を定義するのに対し、イベントストーミングは「裏で何が起きているか」を明らかにする。

---

## 基本的な要素

| 要素 | 色（付箋） | 説明 | 命名規則 |
|------|----------|------|---------|
| **ドメインイベント** | オレンジ | 起きた出来事 | 過去形（OrderPlaced） |
| **コマンド** | 青 | イベントを引き起こす指示 | 命令形（PlaceOrder） |
| **アクター** | 黄 | コマンドを発行する人・システム | 名詞（User, System） |
| **集約** | 黄（大） | コマンドを受け取りイベントを発行する | 名詞（Order） |
| **ポリシー** | 紫 | イベント発生時に自動実行されるルール | 「〜したとき〜する」 |
| **外部システム** | ピンク | システム外との接点 | 名詞（決済サービス, プリンター） |
| **問題・疑問** | 赤 | 未解決の問題や疑問 | 自由 |

---

## 進め方

### ステップ1: ドメインイベントを洗い出す（発散）

「このシステムで何が起きるか？」を自由に出す。時系列順に並べる。

```
例:
OrderPlaced → PaymentReceived → OrderConfirmed
→ KitchenNotified → DishPrepared → OrderServed
```

**ポイント:** 正しさより網羅性を優先する。最初は重複・矛盾があって構わない。

### ステップ2: コマンドを追加する

各イベントの前に「何がトリガーになったか」を追加する。

```
[PlaceOrder] → OrderPlaced
[PrepareDish] → DishPrepared
[ProcessPayment] → PaymentCompleted
```

### ステップ3: アクターを追加する

誰がコマンドを発行するかを追加する。

```
ウェイター → [PlaceOrder] → OrderPlaced
キッチン → [PrepareDish] → DishPrepared
```

### ステップ4: ポリシーを追加する

「イベントAが起きたとき、自動的にコマンドBが実行される」という関係を追加する。

```
OrderConfirmed
  → ポリシー: 注文確定時にキッチンへ自動通知
  → [NotifyKitchen] → KitchenNotified
```

### ステップ5: 集約を特定する

コマンドを受け取ってイベントを発行するオブジェクトをグループ化する。

```
Order集約:
  [PlaceOrder] → OrderPlaced
  [ConfirmOrder] → OrderConfirmed
  [ServeOrder] → OrderServed
```

### ステップ6: 問題・疑問を記録する

進行中に出てきた疑問・未解決の問題を赤付箋で残す。

```
例:
  「注文後に品切れが判明した場合どうするか？」
  「テーブルの移動が発生したら注文はどうなるか？」
```

### ステップ7: 収束させる

- 重複したイベントをまとめる
- 問題・疑問を解決する
- 集約の境界を確定する

---

## コーディングAIとの組み合わせ方

イベントストーミングの各成果物がコーディングAIへの入力になる。

```
ドメインイベント一覧
  → commands_and_events.md に記録
  → Gherkinシナリオの Then として使う

コマンド一覧
  → commands_and_events.md に記録
  → Gherkinシナリオの When として使う

集約の境界
  → domain_model.md に記録
  → 「このコンテキストだけ渡す」単位になる
```

### AIとのイベントストーミング

一人でやるとき、またはリモートでやるときはAIと壁打ちしながら進められる。

```
このシステムで起きうるドメインイベントをすべて洗い出してください。
 以下の振る舞いシナリオを参考にしてください: [シナリオ]

→ AIがドメインイベント候補を列挙
→ 人間が判断・修正・追加
→ 繰り返す
```

---

## 規模別の実施方針

| 規模 | 実施方針 |
|------|---------|
| 1人・単機能 | コマンドとイベントの一覧を簡易的に書くだけで十分 |
| 小チーム・複数機能 | 複雑な箇所だけイベントストーミングを実施 |
| 中規模・複数チーム | 全体をイベントストーミングで発見し、集約の境界を確定する |
| 大規模・複雑ドメイン | 複数回に分けて実施。コンテキストマップまで作成する |

---

## よくある失敗と対策

| 失敗 | 対策 |
|------|------|
| 完璧なモデルを作ろうとして止まる | 1周目は粗くて良い。サイクルを回して育てる |
| 技術的な話に引っ張られる | 「何が起きるか」に集中。「どう実装するか」は後 |
| イベントが動詞になる | 過去形で書く（「注文する」→「OrderPlaced」） |
| 問題を解決しようとして止まる | 赤付箋で記録して先に進む |

---

## 参考文献

### イベントストーミングの原典

- [Alberto Brandolini, "Introducing EventStorming" (2021)](https://www.eventstorming.com/book/) — イベントストーミングの提唱者による公式書籍。Big Picture・Process Level・Software Designの3レベルを解説
- [EventStorming.com](https://www.eventstorming.com/) — Alberto Brandoliniの公式サイト。イベントストーミングの概要・ブログ記事・リソース

### 実践ガイド・事例

- [Mariusz Gil, "Awesome EventStorming"](https://github.com/mariuszgil/awesome-eventstorming) — イベントストーミングのリソース集。スライド・記事・動画・ツールのキュレーション
- [Nick Tune, "EventStorming Tips"](https://medium.com/nick-tune-tech-strategy-blog) — イベントストーミングの実践的なTipsとパターン。ファシリテーションの知見
- [Miro EventStorming Template](https://miro.com/templates/event-storming/) — Miroのイベントストーミングテンプレート。リモートでの実施に活用できる

### DDDとの統合

- [Alberto Brandolini, "EventStorming and Domain-Driven Design"](https://www.eventstorming.com/) — イベントストーミングからDDD成果物への変換プロセス
- [Vaughn Vernon, "Domain-Driven Design Distilled" (2016)](https://www.informit.com/store/domain-driven-design-distilled-9780134434421) — DDDの文脈でのイベントストーミングの位置づけ
- [Nick Tune & Eduardo da Silva, "Architecture Modernization" (2024)](https://www.manning.com/books/architecture-modernization) — イベントストーミングを活用したアーキテクチャ発見・近代化の実践
