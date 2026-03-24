# 第3章: 要求定義を整える

## この章のゴール

> **DDD成果物を含む要求定義が揃っている状態**

第1章・第2章の成果を要求定義のフォーマットに落とす。
チュートリアル1で作った overview.md · features/ · domain/ · adr/ に加え、
**commands_and_events.md と domain_model.md** が新たに登場する。

```
documents/requirements/
├── overview.md
├── features/
│   ├── 01_order.feature
│   ├── 02_kitchen.feature
│   ├── 03_serving.feature
│   └── 04_billing.feature
├── domain/
│   ├── ubiquitous_language.md
│   ├── commands_and_events.md    ← NEW
│   └── domain_model.md           ← NEW
└── adr/
    ├── ADR-001_frontend_backend_split.md
    └── ADR-002_realtime_notification.md
```

---

## チュートリアル1との違い

| 成果物 | チュートリアル1 | チュートリアル2 |
|--------|--------------|--------------|
| overview.md | 作った | 作る（より複雑） |
| features/ | 1ファイル | **複数ファイル**（機能ごとに分割） |
| ubiquitous_language.md | 作った | 作る（アクター・状態が増える） |
| commands_and_events.md | なし | **新規作成** |
| domain_model.md | なし | **新規作成** |
| adr/ | 作った | 作る |

---

## ステップ1: プロダクト要件概要を作る（overview.md）

チュートリアル1と同じ要領で、第1章の要件をまとめる。
複数アクターがいるため、**アクター別に機能を整理する**のがポイント。

### AIへの依頼

```
第1章で整理した要件をもとに仕様書を作ってください。
アクター別（サーバー・厨房・システム）に機能を整理してください。
```

### 成果物のイメージ: overview.md

```markdown
# レストラン注文システム 仕様書

## 概要
テーブルごとに注文を管理し、サーバー・厨房が連携するシステム。

## アクターと画面

| アクター | 画面 | 主な操作 |
|---------|------|---------|
| サーバー | テーブル一覧 | テーブル状態の確認 |
| サーバー | 注文画面 | 注文入力・追加注文・キャンセル |
| サーバー | 配膳画面 | 配膳済み記録 |
| 厨房 | 厨房画面 | 調理完了記録 |

## 機能要件

### サーバー機能
- テーブルを選択して注文を入力できる
- メニューリストから品目を選択する
- 追加注文ができる
- 厨房受付前の品目をキャンセルできる
- 調理完了した品目を配膳済みにできる
- テーブル単位で会計できる

### 厨房機能
- 注文された品目が一覧表示される
- 品目の調理完了を記録できる

### システム機能（自動）
- 調理完了時にサーバーへ通知する（ポリシー）
- 全品配膳済み時にサーバーへ通知する（ポリシー）
- 会計完了時にテーブルを空席に戻す（ポリシー）

## スコープ外
（第1章で整理したものを記載）
```

---

## ステップ2: featureファイルを作る（features/）

チュートリアル1では1ファイルだったが、今回は **機能ごとにファイルを分割** する。

> `guide/appendix/A_bdd.md` のファイル構成ルール:
> 「番号をつけて処理の流れ順に並べると読みやすい」

### AIへの依頼

```
第1章の正常系シナリオと第2章の異常系シナリオを、
正式なGherkin形式のfeatureファイルに整理してください。

以下のファイルに分割してください。
- 01_order.feature（注文の作成・追加・キャンセル）
- 02_kitchen.feature（厨房での調理完了記録）
- 03_serving.feature（配膳）
- 04_billing.feature（会計）

正常系・異常系は同じファイルにまとめてください。
```

### 成果物のイメージ: features/01_order.feature

```gherkin
Feature: 注文管理

  サーバーがテーブルの注文を管理する。

  # --- 正常系 ---

  Scenario: テーブルの注文を入力する
    Given テーブルT1が空席状態である
    When サーバーがT1を選択し「カレー」「サラダ」を注文する
    Then T1の注文に「カレー」「サラダ」が追加される
    And 厨房画面に「T1: カレー」「T1: サラダ」が表示される

  Scenario: 追加注文をする
    Given テーブルT1に「カレー」の注文がある
    When サーバーがT1を選択し「ドリンク」を追加注文する
    Then T1の注文に「ドリンク」が追加される
    And 厨房画面に「T1: ドリンク」が表示される

  Scenario: 厨房受付前に注文品目をキャンセルする
    Given テーブルT1に「サラダ」の注文があり、まだ調理が開始されていない
    When サーバーが「サラダ」をキャンセルする
    Then T1の注文から「サラダ」が消える
    And 厨房画面から「T1: サラダ」が消える

  # --- 異常系 ---

  Scenario: 品切れの品目を注文しようとする
    Given テーブルT1が空席状態である
    And 「カレー」が品切れ状態である
    When サーバーがT1の注文で「カレー」を選択しようとする
    Then 「品切れです」と表示される
    And 「カレー」は注文に追加されない

  Scenario: 調理完了した品目はキャンセルできない
    Given テーブルT1に「カレー」の注文があり、調理が完了している
    When サーバーが「カレー」をキャンセルしようとする
    Then キャンセルは行われない
    And 「調理完了済みのためキャンセルできません」と表示される
```

### featureファイル全体の構成

| ファイル | 正常系 | 異常系 | 合計 |
|---------|--------|--------|------|
| 01_order.feature | 3本 | 2本 | 5本 |
| 02_kitchen.feature | 1本 | 0本 | 1本 |
| 03_serving.feature | 1本 | 1本 | 2本 |
| 04_billing.feature | 1本 | 1本 | 2本 |
| **合計** | **6本** | **4本** | **10本** |

---

## ステップ3: commands_and_events.md を作る（domain/）

第2章のイベントストーミングの成果をフォーマットに落とす。
**これがチュートリアル2で初めて作るDDD成果物**である。

> 書き方の詳細は `guide/appendix/B_ddd.md` を参照。

### AIへの依頼

```
第2章のイベントストーミングの成果をもとに、
commands_and_events.md を作ってください。

以下を含めてください。
- コマンド一覧（トリガーとなるアクター付き）
- ドメインイベント一覧（コマンドとの対応付き）
- ポリシー一覧（トリガーイベントと実行内容）

guide/appendix/B_ddd.md のフォーマットに従ってください。
```

### 成果物のイメージ: domain/commands_and_events.md

```markdown
# コマンドとドメインイベント

## コマンド一覧

| コマンド | アクター | 説明 |
|---------|---------|------|
| **PlaceOrder** | Server | テーブルに新しい注文を作成する |
| **AddItemToOrder** | Server | 既存の注文に品目を追加する |
| **CancelOrderItem** | Server | 注文品目をキャンセルする（調理完了前のみ） |
| **MarkItemCooked** | Kitchen | 品目の調理完了を記録する |
| **MarkItemServed** | Server | 品目の配膳済みを記録する |
| **RequestBill** | Server | テーブルの会計を依頼する |
| **SettleBill** | Server | 会計を完了する |
| **ClearTable** | System | テーブルを空席状態に戻す |

## ドメインイベント一覧

### PlaceOrder フロー
PlaceOrder
  └─→ OrderPlaced （注文が作成された）

### AddItemToOrder フロー
AddItemToOrder
  └─→ ItemAddedToOrder （品目が注文に追加された）

### CancelOrderItem フロー
CancelOrderItem
  └─→ OrderItemCancelled （注文品目がキャンセルされた）

### MarkItemCooked フロー
MarkItemCooked
  └─→ ItemCooked （品目の調理が完了した）
       └─→ [ポリシー] サーバーに配膳通知

### MarkItemServed フロー
MarkItemServed
  └─→ ItemServed （品目が配膳された）
       └─→ [ポリシー] 全品配膳済みなら会計可能通知

### SettleBill フロー
SettleBill
  └─→ BillSettled （会計が完了した）
       └─→ [ポリシー] テーブルクリア

## ポリシー（自動実行ルール）

| トリガーイベント | ポリシー | 実行内容 |
|----------------|---------|---------|
| ItemCooked | 調理完了通知 | サーバーの配膳画面に通知する |
| ItemServed（全品配膳済みの場合） | 全品配膳済み通知 | サーバーに会計可能を通知する |
| BillSettled | テーブルクリア | テーブルを空席状態に戻す |
```

---

## ステップ4: domain_model.md を作る（domain/）

エンティティ・値オブジェクト・状態遷移を定義する。

### AIへの依頼

```
ここまでの仕様書・コマンド・イベントをもとに、
domain_model.md を作ってください。

以下を含めてください。
- エンティティと値オブジェクトの定義
- 注文品目の状態遷移
- テーブルの状態遷移

guide/appendix/B_ddd.md のフォーマットに従ってください。
```

### 成果物のイメージ: domain/domain_model.md

```markdown
# ドメインモデル

## エンティティ

### Table（テーブル）
テーブル。固定4卓（T1〜T4）。

​```
Table
  - id: TableId          # T1, T2, T3, T4
  - status: TableStatus  # empty / occupied / billing
  - currentOrder: Order? # 現在の注文（空席時はnull）
​```

### Order（注文）
テーブル単位の注文。1テーブルにつき同時に1つ。

​```
Order
  - id: OrderId
  - tableId: TableId
  - items: []OrderItem
  - createdAt: DateTime
​```

### OrderItem（注文品目）
注文内の1品目。状態を持つ。

​```
OrderItem
  - id: OrderItemId
  - menuItemId: MenuItemId
  - name: string
  - price: number
  - status: OrderItemStatus  # ordered / cooked / served / cancelled
​```

## 値オブジェクト

### MenuItem（メニュー品目）
メニューの1品目。固定リスト。

​```
MenuItem
  - id: MenuItemId
  - name: string
  - price: number
  - available: boolean  # 品切れフラグ
​```

## 状態遷移

### OrderItem のステータス

​```
ordered → cooked → served
    │
    └→ cancelled（ordered からのみ遷移可能）
​```

### Table のステータス

​```
empty → occupied（注文が入ったとき）
occupied → billing（会計依頼時）
billing → empty（会計完了時 ← ポリシー: テーブルクリア）
​```
```

### 人間がレビューすること

- 状態遷移に矛盾がないか（cancelledに遷移できるのはorderedからのみ、が正しいか）
- featureファイルのシナリオと整合しているか
- ユビキタス言語と命名が一致しているか

---

## ステップ5: ユビキタス言語を更新する（domain/）

チュートリアル1と同様に、プロジェクト内の言葉を統一する。
今回はアクター・状態・イベント名が加わるため、語彙が大きくなる。

### AIへの依頼

```
ここまでの成果物（spec, features, commands_and_events, domain_model）に
登場する用語を整理して、ユビキタス言語の定義一覧を作ってください。
```

### 成果物のイメージ（抜粋）

```markdown
# ユビキタス言語

## アクター

| 用語 | 定義 |
|------|------|
| **サーバー（Server）** | 注文入力・配膳・会計を行う給仕スタッフ。お客様の操作を代行する |
| **厨房（Kitchen）** | 注文品目を調理し、完了を記録するスタッフ |

## テーブル関連

| 用語 | 定義 |
|------|------|
| **テーブル（Table）** | 座席の単位。T1〜T4の固定4卓 |
| **空席（Empty）** | 注文がないテーブルの状態 |
| **使用中（Occupied）** | 注文があるテーブルの状態 |
| **会計中（Billing）** | 会計処理中のテーブルの状態 |

## 注文関連

| 用語 | 定義 |
|------|------|
| **注文（Order）** | テーブル単位の注文。複数の品目を含む |
| **注文品目（OrderItem）** | 注文内の1品。メニュー品目・状態・金額を持つ |
| **品切れ（Unavailable）** | メニュー品目が注文できない状態 |

## 品目の状態

| 用語 | 定義 |
|------|------|
| **注文済み（Ordered）** | 注文されたがまだ調理されていない状態 |
| **調理完了（Cooked）** | 厨房で調理が完了し、配膳を待つ状態 |
| **配膳済み（Served）** | テーブルに届けられた状態 |
| **キャンセル済み（Cancelled）** | 注文済み状態からキャンセルされた状態 |
```

---

## ステップ6: ADRを書く（adr/）

このチュートリアルでは2つの技術的判断をADRとして記録する。

### ADR候補

| ADR | 判断内容 |
|-----|---------|
| ADR-001 | フロントエンド・バックエンドを分離する（REST API） |
| ADR-002 | リアルタイム通知の方式（ポーリング vs WebSocket） |

### AIへの依頼

```
以下の技術的判断についてADRを書いてください。

判断1: フロントエンドとバックエンドを分離し、REST APIで通信する。
  理由: 複数画面（サーバー画面・厨房画面）が同じデータにアクセスする必要がある。
  チュートリアル1のlocalStorageでは画面間のデータ共有ができない。

判断2: リアルタイム通知はポーリング（定期的なAPI呼び出し）で実現する。
  理由: WebSocketはインフラが複雑になる。チュートリアルではポーリングで十分。
```

> チュートリアル1との違いとして、バックエンドが登場する。
> これは複数アクター（サーバー画面・厨房画面）がデータを共有するためである。

---

## この章の成果物

```
documents/requirements/
├── overview.md                            ✅ プロダクト要件概要
├── features/
│   ├── 01_order.feature                   ✅ 注文（正常系3 + 異常系2）
│   ├── 02_kitchen.feature                 ✅ 厨房（正常系1）
│   ├── 03_serving.feature                 ✅ 配膳（正常系1 + 異常系1）
│   └── 04_billing.feature                 ✅ 会計（正常系1 + 異常系1）
├── domain/
│   ├── ubiquitous_language.md             ✅ ユビキタス言語
│   ├── commands_and_events.md             ✅ コマンド・イベント・ポリシー
│   └── domain_model.md                    ✅ エンティティ・状態遷移
└── adr/
    ├── ADR-001_frontend_backend_split.md  ✅ API分離
    └── ADR-002_realtime_notification.md   ✅ 通知方式
```

---

## 振り返り

- **DDD成果物の価値:** commands_and_events.md と domain_model.md を作ることで、
  「AIに渡す設計情報」の精度が格段に上がった
- **featureファイルの分割:** 機能ごとにファイルを分けることで、
  「この機能だけ渡して実装させる」が可能になった
- **イベントストーミングからの自然な流れ:** 第2章で発見した構造が、
  そのままドキュメントの構造になった

---

## 次の章へ

第4章では、この要求定義をAIに渡して
詳細設計の生成 → 実装を行う。

チュートリアル1と同じ流れだが、今回は **異常系シナリオを受け入れ条件に含める** 点が新しい。
