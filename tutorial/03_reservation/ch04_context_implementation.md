# 第4章: コンテキスト単位で詳細設計を生成・実装する

## この章のゴール

> **各コンテキストのシナリオを満たす動くアプリケーションがある状態**

3つのコンテキストを順番にAIに委譲して実装する。
**コンテキスト単位でコーディングAIに渡す** 体験がこの章の核心。

---

## なぜコンテキスト単位で委譲するのか

`guide/05_bdd_and_ddd.md` より:

> 大きな仕様書を丸ごと渡すより、**境界付けられたコンテキスト単位で渡す**方が品質が安定する。

理由は3つ。

1. **コンテキストが小さい:** AIが処理する情報量が絞られ、精度が上がる
2. **関心が明確:** 「予約の整合性」「顧客の管理」など、1つのコンテキストは1つの関心に集中している
3. **レビューしやすい:** コンテキスト単位で実装→レビュー→次のコンテキストと進められる

> これは `guide/appendix/F_context_engineering.md` の「スコープ限定パターン」の実践である。
> 必要な情報だけを渡し、作業範囲を明示的に限定することでAIの精度を上げる。
>
> 実装委譲時のリンター・Hooks・指示ファイル等の実行環境設計（ハーネスエンジニアリング）については
> `guide/appendix/I_harness_engineering.md` を参照。

```
❌ 全部まとめて渡す:
  「reservation/ + staff/ + customer/ + adr/ を全部読んで実装してください」
  → コンテキストが混在し、AIが関心を見失いやすい

✅ コンテキスト単位で渡す:
  「reservation/ と adr/ を読んで、予約コンテキストだけ実装してください」
  → AIが予約の関心に集中できる
```

---

## 実装の順序

コンテキスト間の依存関係を考慮して、以下の順序で進める。

```
① スタッフコンテキスト（依存なし。最もシンプル）
② 顧客コンテキスト（依存なし）
③ 予約コンテキスト（①②に依存。最も複雑）
```

> 予約コンテキストはスタッフと顧客の存在確認が必要なため、
> 先に①②を実装しておく。

---

## ステップ1: スタッフコンテキストの実装

### 詳細設計の生成

```
以下のファイルを読んで、スタッフコンテキストの specification/ を生成してください。

- documents/requirements/staff/overview.md
- documents/requirements/staff/domain/ubiquitous_language.md
- documents/requirements/staff/domain/domain_model.md
- documents/requirements/adr/ADR-003_monolithic.md
- documents/requirements/adr/ADR-004_sqlite.md

documents/specification/staff/ に以下を作ってください。
- architecture.md
- backend/api_design.md
```

### 実装

```
documents/specification/staff/ を読んで、
implement/ にスタッフコンテキストを実装してください。

モジュール構成は以下の通りです。
implement/src/staff/  （または言語に応じたパッケージ構成）

スタッフは固定データ（A, B, C）で構いません。
APIとして GET /api/staff（一覧取得）を実装してください。
```

### 確認

- APIが正しく動作するか
- スタッフ3人の情報が取得できるか

> スタッフコンテキストはシンプルなため、素早く完了するはず。
> これが「コンテキスト単位の委譲」のウォームアップになる。

---

## ステップ2: 顧客コンテキストの実装

### 詳細設計の生成

```
以下のファイルを読んで、顧客コンテキストの specification/ を生成してください。

- documents/requirements/customer/overview.md
- documents/requirements/customer/features/01_customer.feature
- documents/requirements/customer/domain/ubiquitous_language.md
- documents/requirements/customer/domain/domain_model.md
- documents/requirements/adr/ADR-003_monolithic.md
- documents/requirements/adr/ADR-004_sqlite.md

documents/specification/customer/ に以下を作ってください。
- architecture.md
- backend/api_design.md
```

### 実装

```
documents/specification/customer/ を読んで、
顧客コンテキストを実装してください。

以下のシナリオを受け入れ条件として満たしてください。

Scenario: 顧客を登録する
  When 名前「山田太郎」、電話番号「090-1234-5678」で顧客を登録する
  Then 顧客リストに「山田太郎」が追加される
```

### 確認

- 顧客の登録・取得ができるか
- featureファイルのシナリオが満たされているか

---

## ステップ3: 予約コンテキストの実装

ここが最も複雑で、コンテキスト単位委譲の真価が問われる。

### 詳細設計の生成

```
以下のファイルを読んで、予約コンテキストの specification/ を生成してください。

- documents/requirements/reservation/overview.md
- documents/requirements/reservation/features/01_reservation.feature
- documents/requirements/reservation/features/02_waiting_list.feature
- documents/requirements/reservation/domain/ubiquitous_language.md
- documents/requirements/reservation/domain/commands_and_events.md
- documents/requirements/reservation/domain/domain_model.md
- documents/requirements/adr/（全ADR）

documents/specification/reservation/ に以下を作ってください。
- architecture.md
- backend/api_design.md（REST APIエンドポイント）
- backend/sequences/（予約作成・キャンセル・キャンセル待ち昇格のフロー）

特に以下を重視してください。
- ADR-001の楽観的ロックを反映する
- ADR-002のコンテキスト間通信（同期）を反映する
- domain_model.md のTimeSlot集約による二重予約防止を設計に反映する
```

### 詳細設計のレビュー観点

| 観点 | 確認内容 |
|------|---------|
| 集約の保護 | TimeSlot集約の外から直接予約を作れない設計になっているか |
| 楽観的ロック | バージョン番号のチェックがAPIフローに含まれているか |
| ポリシー | キャンセル待ち昇格が cancel のフローに含まれているか |
| コンテキスト間通信 | スタッフ/顧客の確認がインターフェース経由になっているか |

### 実装（段階的に）

予約コンテキストは複雑なため、さらに分割して依頼する。

#### ③-a: 予約作成

```
予約コンテキストの予約作成機能を実装してください。

以下のシナリオを受け入れ条件として満たしてください。

Scenario: 予約を作成する
  Given スタッフAの10:00枠が空いている
  When 顧客「山田太郎」がスタッフAの10:00に「カット」を予約する
  Then スタッフAの10:00に「山田太郎 - カット」の予約が存在する
  And スタッフAの10:00枠が予約済みになる

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

ADR-001（楽観的ロック）に従ってください。
documents/specification/reservation/backend/sequences/ を参照してください。
```

#### ③-b: キャンセル

```
予約キャンセル機能を実装してください。

Scenario: 予約をキャンセルする
  Given スタッフAの10:00に「山田太郎」の予約がある
  When 「山田太郎」の予約をキャンセルする
  Then スタッフAの10:00枠が空きに戻る

キャンセル待ち昇格はまだ実装しないでください（③-cで実装します）。
```

#### ③-c: キャンセル待ちと昇格ポリシー

```
キャンセル待ちと昇格ポリシーを実装してください。

以下のシナリオをすべて受け入れ条件として満たしてください。

Scenario: キャンセル待ちに登録する
  ...

Scenario: キャンセル待ちが自動昇格する
  ...

Scenario: キャンセル待ちがいない場合は昇格しない
  ...

③-bのキャンセル処理に昇格ポリシーを組み込んでください。
commands_and_events.md のポリシー定義を参照してください。
```

---

## ステップ4: フロントエンドを実装する

バックエンドの3コンテキストが完成したら、フロントエンドを実装する。

### 画面構成

```
予約画面（メイン）
  └─ スタッフ × 時間帯のグリッド表示
  └─ 空き枠をクリック → 予約作成ダイアログ
  └─ 予約済み枠をクリック → 詳細表示（キャンセルボタン）
  └─ 予約済み枠 → キャンセル待ち登録ボタン

顧客管理画面
  └─ 顧客の登録・一覧表示
```

### AIへの依頼

```
documents/specification/ を参照して、
フロントエンドを実装してください。

チュートリアル2と同様に、バックエンドのAPIを呼び出す形にしてください。
Page Objectパターンも導入してください（チュートリアル2で学んだ手法）。
```

---

## ステップ5: 全シナリオを確認する

### 予約コンテキスト

| # | シナリオ | 種別 | 結果 |
|---|---------|------|------|
| 1 | 予約を作成する | 正常系 | |
| 2 | 予約をキャンセルする | 正常系 | |
| 3 | 同一スタッフの同一時間帯に二重予約できない | 異常系 | |
| 4 | 同一顧客が同一時間帯に複数予約できない | 異常系 | |
| 5 | キャンセル待ちに登録する | 正常系 | |
| 6 | キャンセル待ちが自動昇格する | 正常系 | |
| 7 | キャンセル待ちがいない場合は昇格しない | 正常系 | |

### 顧客コンテキスト

| # | シナリオ | 種別 | 結果 |
|---|---------|------|------|
| 8 | 顧客を登録する | 正常系 | |

---

## この章の成果物

```
documents/specification/
├── staff/          ✅ スタッフコンテキスト詳細設計
├── customer/       ✅ 顧客コンテキスト詳細設計
└── reservation/    ✅ 予約コンテキスト詳細設計

implement/
├── src/
│   ├── staff/        ✅ スタッフモジュール
│   ├── customer/     ✅ 顧客モジュール
│   ├── reservation/  ✅ 予約モジュール（集約・楽観的ロック）
│   └── frontend/     ✅ フロントエンド
├── （設定ファイル群）
└── database.sqlite   ✅ SQLiteデータベース
```

---

## 振り返り

- **コンテキスト単位の委譲が機能した:** 1つのコンテキストだけ渡すことで、
  AIが「今何に集中すべきか」を迷わなかった
- **依存順序の重要性:** スタッフ → 顧客 → 予約の順で実装することで、
  予約コンテキストが他を参照できる状態で着手できた
- **予約コンテキストのさらなる分割:** 最も複雑なコンテキストも、
  予約作成 → キャンセル → キャンセル待ちと段階的に進めることで管理できた
- **ADRの効果:** 詳細設計の生成時にADRを一緒に渡すことで、
  楽観的ロックや同期通信が自然に設計に反映された

### コンテキストエンジニアリングのパターンとの対応

この章で実践したことは、`guide/appendix/F_context_engineering.md` のパターンと直接対応している。

| この章で実践したこと | 対応するパターン |
|-------------------|---------------|
| コンテキストごとに必要なファイルだけ列挙して渡した | **ポインタ渡し** — ファイルパスだけを渡し、AIに読ませる |
| 「予約コンテキストだけ実装してください」と明示した | **スコープ限定** — 作業範囲を明示的に限定する |
| フェーズ1の成果物をフェーズ2に渡し、フェーズ2の結果をフェーズ3に渡した | **ブリッジコンテキスト** — フェーズ間は「結論だけ」を渡す |
| 予約コンテキストをさらに③-a/③-b/③-cに分割して渡した | **スコープ限定**の入れ子適用 |

---

## 次の章へ

第5章では、二重予約防止やキャンセル待ち昇格などの
複雑なドメインルールをコンポーネントテストでカバーする。
