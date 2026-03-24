# 第2章: コンテキストごとに要求定義を作る

## この章のゴール

> **各コンテキストの要求定義が揃っている状態**

第1章で識別した3つのコンテキストごとに、
ubiquitous_language / commands_and_events / domain_model を作る。

特に予約コンテキストの **集約の境界** と **競合制御（二重予約防止）** の設計が核心。

---

## ディレクトリ構成

コンテキストごとに documents/ を分ける。

```
documents/requirements/
├── reservation/                # 予約コンテキスト
│   ├── overview.md
│   ├── features/
│   │   ├── 01_reservation.feature
│   │   └── 02_waiting_list.feature
│   └── domain/
│       ├── ubiquitous_language.md
│       ├── commands_and_events.md
│       └── domain_model.md
├── staff/                      # スタッフコンテキスト
│   ├── overview.md
│   └── domain/
│       ├── ubiquitous_language.md
│       └── domain_model.md
├── customer/                   # 顧客コンテキスト
│   ├── overview.md
│   ├── features/
│   │   └── 01_customer.feature
│   └── domain/
│       ├── ubiquitous_language.md
│       └── domain_model.md
└── adr/                        # ADR（コンテキスト横断）
    └── （第3章で作成）
```

> **ポイント:** ADRはコンテキストをまたぐ判断が多いため、
> コンテキストの外に置く。

---

## チュートリアル2との違い

| 観点 | チュートリアル2 | チュートリアル3 |
|------|--------------|--------------|
| documents/ の構成 | フラット | **コンテキストごとに分割** |
| domain_model.md | 1ファイル | **コンテキストごとに1ファイル** |
| 集約 | 暗黙的 | **明示的に設計** |
| 競合制御 | なし | **二重予約防止** |

---

## ステップ1: 予約コンテキストを設計する

予約コンテキストが最も複雑で、このチュートリアルの核心。

### プロダクト要件概要: reservation/overview.md

```markdown
# 予約コンテキスト 仕様書

## 概要
スタッフ × 時間帯の予約枠を管理する。
同一スタッフの同一時間帯に二重予約できないことを保証する。

## 機能要件
- 予約を作成できる（スタッフ × 時間帯 × 顧客 × メニュー）
- 予約をキャンセルできる
- キャンセル待ちに登録できる
- キャンセル時にキャンセル待ちの先頭が自動昇格する（ポリシー）
- 予約可能な枠を一覧表示できる

## 制約
- 同一スタッフの同一時間帯に2つ以上の予約は存在できない
- 同一顧客が同一時間帯に複数の予約はできない
- キャンセル待ちからの昇格はFIFO（先着順）
```

### featureファイル: reservation/features/01_reservation.feature

```gherkin
Feature: 予約管理

  スタッフ × 時間帯の予約を管理する。

  # --- 正常系 ---

  Scenario: 予約を作成する
    Given スタッフAの10:00枠が空いている
    When 顧客「山田太郎」がスタッフAの10:00に「カット」を予約する
    Then スタッフAの10:00に「山田太郎 - カット」の予約が存在する
    And スタッフAの10:00枠が予約済みになる

  Scenario: 予約をキャンセルする
    Given スタッフAの10:00に「山田太郎」の予約がある
    When 「山田太郎」の予約をキャンセルする
    Then スタッフAの10:00枠が空きに戻る

  # --- 異常系: 二重予約防止 ---

  Scenario: 同一スタッフの同一時間帯に二重予約できない
    Given スタッフAの10:00に「山田太郎」の予約がある
    When 顧客「佐藤花子」がスタッフAの10:00に予約しようとする
    Then 予約は作成されない
    And 「この枠はすでに予約済みです」と表示される

  Scenario: 同一顧客が同一時間帯に複数予約できない
    Given 顧客「山田太郎」がスタッフAの10:00に予約している
    When 顧客「山田太郎」がスタッフBの10:00に予約しようとする
    Then 予約は作成されない
    And 「同じ時間帯にすでに予約があります」と表示される
```

### featureファイル: reservation/features/02_waiting_list.feature

```gherkin
Feature: キャンセル待ち

  予約済みの枠に対してキャンセル待ちを管理する。

  Scenario: キャンセル待ちに登録する
    Given スタッフAの10:00枠が予約済みである
    When 顧客「佐藤花子」がスタッフAの10:00のキャンセル待ちに登録する
    Then キャンセル待ちリストに「佐藤花子」が追加される

  Scenario: キャンセル待ちが自動昇格する
    Given スタッフAの10:00に「山田太郎」の予約がある
    And キャンセル待ちに「佐藤花子」「鈴木一郎」が登録されている
    When 「山田太郎」の予約をキャンセルする
    Then スタッフAの10:00に「佐藤花子」の予約が自動作成される
    And キャンセル待ちリストから「佐藤花子」が消える
    And 「鈴木一郎」はキャンセル待ちに残る

  Scenario: キャンセル待ちがいない場合は昇格しない
    Given スタッフAの10:00に「山田太郎」の予約がある
    And キャンセル待ちリストが空である
    When 「山田太郎」の予約をキャンセルする
    Then スタッフAの10:00枠が空きに戻る
```

### domain_model.md: 集約と競合制御

ここが最も重要。

### AIへの依頼

```
予約コンテキストの domain_model.md を作ってください。
以下を含めてください。

1. 集約の境界: 何が集約ルートで、何が集約の中にいるか
2. 競合制御: 二重予約防止をドメインモデルでどう表現するか
3. 状態遷移: TimeSlot の状態遷移

制約:
- 同一スタッフの同一時間帯に2つ以上の予約は存在できない
- この制約は集約の境界で保護する（集約の外からは破れない）
```

### 成果物のイメージ: reservation/domain/domain_model.md

```markdown
# ドメインモデル（予約コンテキスト）

## 集約: TimeSlot（時間枠）

TimeSlotが集約ルート。
「スタッフ × 時間帯」単位で予約の整合性を保証する。

​```
TimeSlot（集約ルート）
  - staffId: StaffId
  - dateTime: DateTime          # 日付 + 時間帯（例: 2024-01-15 10:00）
  - reservation: Reservation?   # 予約（最大1つ）
  - waitingList: []WaitingEntry # キャンセル待ちリスト

  + reserve(customerId, menuId): void
      → ReservationCreated
      ※ すでに予約がある場合は例外
  + cancel(): void
      → ReservationCancelled
      → [ポリシー] キャンセル待ち昇格
  + joinWaitingList(customerId): void
      → WaitingListJoined
  + promoteFromWaitingList(): void
      → WaitingListPromoted / ReservationCreated
​```

## 二重予約防止の仕組み

​```
TimeSlot が reservation を最大1つしか持たない
  → reserve() 時に reservation が null かチェック
  → null でなければ例外（二重予約防止）

この制約は TimeSlot 集約の内部で保証される。
集約の外から直接 reservation を設定する手段はない。
​```

## エンティティ

### Reservation（予約）
TimeSlot 内に最大1つ存在する。

​```
Reservation
  - id: ReservationId
  - customerId: CustomerId
  - menuId: MenuId
  - createdAt: DateTime
​```

### WaitingEntry（キャンセル待ちエントリ）
TimeSlot 内のキャンセル待ちリストの1要素。

​```
WaitingEntry
  - customerId: CustomerId
  - joinedAt: DateTime       # FIFO順序に使用
​```

## 状態遷移

### TimeSlotの状態

​```
available（空き）
  → reserved（予約済み）  [reserve() 成功時]

reserved（予約済み）
  → available（空き）     [cancel() かつ キャンセル待ちなし]
  → reserved（予約済み）  [cancel() かつ キャンセル待ちあり → 自動昇格]
​```
```

### 人間がレビューすること

- **集約の境界が正しいか:** TimeSlot が集約ルートである理由は「スタッフ × 時間帯の整合性を保証する最小単位」だから。これに納得できるか
- **二重予約防止が集約の中で閉じているか:** 外部から予約を直接追加できない設計になっているか
- **キャンセル待ち昇格のフロー:** cancel() の中で自動的にpromote() が呼ばれる設計は妥当か

> **なぜ集約が重要か:** 二重予約防止のルールを「APIバリデーション」ではなく
> 「ドメインモデルの構造」で保証する。
> これにより、どの経路からアクセスしても二重予約が起きない。

---

## ステップ2: スタッフコンテキストを設計する

スタッフコンテキストは比較的シンプル。

### 成果物のイメージ: staff/domain/domain_model.md

```markdown
# ドメインモデル（スタッフコンテキスト）

## エンティティ

### Staff（スタッフ）
​```
Staff
  - id: StaffId
  - name: string
​```

### Schedule（スケジュール）
​```
Schedule
  - staffId: StaffId
  - availableSlots: []DateTime   # 稼働可能な時間帯
​```

## 備考
このチュートリアルでは全スタッフが全時間帯で稼働と仮定する。
Schedule は将来のシフト管理のための拡張ポイントとして定義しておく。
```

> スタッフコンテキストにはコマンドがない（固定データ）。
> 予約コンテキストから「staffId で空き確認」を問い合わせるだけ。

---

## ステップ3: 顧客コンテキストを設計する

### 成果物のイメージ: customer/domain/domain_model.md

```markdown
# ドメインモデル（顧客コンテキスト）

## エンティティ

### Customer（顧客）
​```
Customer
  - id: CustomerId
  - name: string
  - contactInfo: ContactInfo
​```

### ContactInfo（連絡先）
​```
ContactInfo
  - phoneNumber: string
​```

## コマンド・イベント

| コマンド | イベント |
|---------|---------|
| RegisterCustomer | CustomerRegistered |
| UpdateCustomer | CustomerUpdated |
```

---

## ステップ4: ユビキタス言語を整理する

### コンテキストごとに「同じ言葉の意味が違う」を明示する

```markdown
# ユビキタス言語（予約コンテキスト）

| 用語 | 定義（予約コンテキストにおける意味） |
|------|------|
| **時間枠（TimeSlot）** | スタッフ × 日時の組み合わせ。予約可能な最小単位。集約ルート |
| **予約（Reservation）** | 時間枠に対する顧客の予約。1つの時間枠に最大1つ |
| **キャンセル待ち（WaitingList）** | 予約済み枠に対する待機リスト。FIFO順 |
| **昇格（Promote）** | キャンセル待ちの先頭が自動的に予約に変わること |

# ユビキタス言語（スタッフコンテキスト）

| 用語 | 定義（スタッフコンテキストにおける意味） |
|------|------|
| **スタッフ（Staff）** | 施術を担当する人。ID・名前を持つ |
| **スケジュール（Schedule）** | スタッフの稼働可能な時間帯の集合 |

# ユビキタス言語（顧客コンテキスト）

| 用語 | 定義（顧客コンテキストにおける意味） |
|------|------|
| **顧客（Customer）** | サービスを受ける人。名前・連絡先を持つ |
```

> **コンテキストをまたいで同じ言葉を使うが意味が違うケース:**
> 「予約」は予約コンテキストでは「時間枠に対する排他的な確保」だが、
> 顧客コンテキストでは「顧客の予約履歴の1件」にすぎない。

---

## この章の成果物

```
documents/requirements/
├── reservation/
│   ├── overview.md                         ✅
│   ├── features/01_reservation.feature     ✅（正常系2 + 異常系2）
│   ├── features/02_waiting_list.feature    ✅（正常系3）
│   └── domain/
│       ├── ubiquitous_language.md          ✅
│       ├── commands_and_events.md          ✅
│       └── domain_model.md                ✅（集約・競合制御）
├── staff/
│   ├── overview.md                         ✅
│   └── domain/
│       ├── ubiquitous_language.md          ✅
│       └── domain_model.md                ✅
├── customer/
│   ├── overview.md                         ✅
│   ├── features/01_customer.feature        ✅
│   └── domain/
│       ├── ubiquitous_language.md          ✅
│       └── domain_model.md                ✅
└── adr/                                    （第3章で作成）
```

---

## 振り返り

- **集約の価値:** 「二重予約できない」というルールが、コード上の if 文ではなく、
  ドメインモデルの構造として表現された
- **コンテキスト分割の価値:** コンテキストごとにドキュメントを分けたことで、
  「このコンテキストだけ渡せばAIに実装を委譲できる」状態になった
- **ユビキタス言語の重要性が増す:** コンテキストが増えると、
  同じ言葉が異なる意味で使われるリスクが高まる。明示的な定義がより重要

---

## 次の章へ

第3章では、コンテキストをまたぐ設計判断をADRとして記録する。
特に「競合制御の実装方針」と「コンテキスト間の通信方式」は
実装の品質に大きく影響する判断である。
