# 第3章: 詳細設計を生成する

## この章のゴール

> **AIが実装に着手できる詳細設計がある状態**

第2章で整えた要求定義をAIに渡して、詳細設計を生成させる。
生成された詳細設計をレビューし、意図との乖離があれば修正する。

```
documents/specification/
├── architecture.md       # パッケージ構成・コンポーネント設計
├── test_plan.md          # テスト方針
└── backend/
    ├── api_design.md     # データ構造・インターフェース定義
    └── sequences/        # 処理フローの詳細
        ├── card_create.md
        ├── card_move.md
        └── card_delete.md
```

---

## フェーズ2の位置づけ

ここからフェーズ2に入る。

```
フェーズ1: 要求定義を育てる（第1章・第2章）← 完了
フェーズ2: 詳細設計を生成する（この章）    ← いまここ
フェーズ3: 実装する（第4章）
```

フェーズ1では人間が主導した。フェーズ2ではAIが生成し、人間がレビューする。
`guide/04_collaboration_matrix.md` でいう「レビュー型」の協調パターンである。

> **注意:** この章からは、ファイルの読み書きができるコーディングAI（Claude Code、Cursor、GitHub Copilot Agent等）の利用を前提とする。
> 第1章・第2章はチャットAIでも進められたが、詳細設計の生成ではプロジェクトのファイルを直接読み書きできる環境が効率的である。
> チャットAIで進める場合は、各ファイルの内容をプロンプトに貼り付ける形でも可能だが、ファイル数が増えるため手間が大きくなる。

---

## ステップ1: AIに詳細設計を生成させる

### AIへの依頼

```
documents/requirements/ 配下のファイルをすべて読んで、
documents/specification/ を生成してください。

以下のファイルを作ってください。
- architecture.md（コンポーネント構成・モジュール依存関係）
- test_plan.md（テスト方針・カバレッジ計画）
- backend/api_design.md（データ構造・インターフェース定義）
- backend/sequences/（主要な処理フローのシーケンス）

ADRに記録された技術的判断（技術選定とその理由）に従ってください。
```

> **ポイント:** 「すべて読んで」と明示する。
> AIがどのファイルを読んだかは出力から確認できる。
> 読み落としがある場合は追加で読ませる。

---

## ステップ2: 生成結果をレビューする

AIが生成した各ファイルについてレビューする。
コーディングAIはファイルを一括で生成してくるため、ファイルごとに観点を持ってレビューすることが重要である。

### architecture.md のレビュー

例えば以下のような内容が生成される。

```markdown
# アーキテクチャ

## 技術スタック
- React 18 + TypeScript
- Vite（ビルドツール）
- localStorage（永続化）

## コンポーネント構成

​```
App
├── Board
│   ├── Column（×3: Todo / InProgress / Done）
│   │   ├── ColumnHeader
│   │   └── Card（×N）
│   └── CardCreateForm
└── （将来的に追加される可能性のあるコンポーネント）
​```

## 状態管理
- useReducer + Context でボード全体の状態を管理
- 状態変更のたびに localStorage に保存

## ディレクトリ構成

​```
implement/
├── src/
│   ├── components/    # UIコンポーネント
│   ├── hooks/         # カスタムフック
│   ├── types/         # 型定義
│   ├── reducers/      # 状態管理ロジック
│   └── storage/       # localStorage操作
├── package.json
├── tsconfig.json
└── vite.config.ts
​```
```

レビューの観点：

- ADRの技術的判断と整合しているか
- 仕様との整合性があるか（3つの固定カラム構成が反映されているか等）
- チュートリアルの規模に対して過剰設計になっていないか
- この設計で全シナリオを満たせるか

### backend/api_design.md のレビュー

例えば以下のような内容が生成される。

```markdown
# データ構造・インターフェース定義

## 型定義

​```typescript
type ColumnId = 'todo' | 'inProgress' | 'done';

interface Card {
  id: string;
  title: string;
  columnId: ColumnId;
  createdAt: string; // ISO 8601
}

interface Board {
  cards: Card[];
}
​```

## 操作インターフェース

​```typescript
// カード作成
createCard(title: string): Card

// カード移動（隣接カラムのみ）
moveCard(cardId: string, direction: 'forward' | 'backward'): void

// カード削除
deleteCard(cardId: string): void

// ボード取得（localStorage から復元）
loadBoard(): Board

// ボード保存（localStorage へ保存）
saveBoard(board: Board): void
​```

## カラムの隣接関係

​```
todo ←→ inProgress ←→ done
​```

moveCard の direction:
- forward: todo → inProgress → done
- backward: done → inProgress → todo
```

レビューの観点：

- ユビキタス言語と命名が一致しているか（Card, Board, Column等）
- 隣接カラムのみ移動できる制約が表現されているか
- 各操作がfeatureファイルのシナリオに対応しているか

### backend/sequences/ のレビュー

例えば以下のような内容が生成される（card_create.md）。

```markdown
# シーケンス: カード作成

​```
ユーザー → CardCreateForm: タイトルを入力
CardCreateForm → CardCreateForm: バリデーション（空文字チェック）
  [空文字の場合]
  CardCreateForm → ユーザー: エラーメッセージ表示
  [正常な場合]
  CardCreateForm → BoardReducer: CREATE_CARD アクション
  BoardReducer → Board状態: カードをTodoに追加
  Board状態 → localStorage: 保存
  Board状態 → Board: 再レンダリング
​```
```

レビューの観点：

- featureファイルの正常系・異常系の両方がフローに含まれているか
- エラー時の振る舞いが仕様と一致しているか

### test_plan.md のレビュー

例えば以下のような内容が生成される。

```markdown
# テスト方針

## テスト種別

| 種別 | 対象 | ツール |
|------|------|--------|
| ユニットテスト | Reducer のロジック | Vitest |
| E2Eテスト | ユーザー操作の全フロー | Playwright |

## カバレッジ方針

### ユニットテスト
- カード作成ロジック（正常系・空文字バリデーション）
- カード移動ロジック（正常系・隣接チェック・飛び越し防止）
- カード削除ロジック
- localStorage の保存・復元

### E2Eテスト
- features/01_card_management.feature の全シナリオ
```

レビューの観点：

- featureファイルの全シナリオがテスト計画に含まれているか
- チュートリアルの規模に対して過剰でないか

---

## ステップ3: 修正を依頼する

レビューで気になった点があれば、具体的に修正を依頼する。

> **ポイント:** 修正には必ず理由を添える。
> AIは「何を」だけでなく「なぜ」が分かると、他の箇所への影響も考慮してくれる。

今回のチュートリアルでは、レビューの結果、以下のような修正を依頼した。

```
architecture.md について以下を修正してください。
- 状態管理は useReducer + Context ではなく useState でシンプルに始めたい。
  理由: チュートリアル1の規模では useReducer は過剰。
- ディレクトリの reducers/ は不要。hooks/ に状態管理を含めてください。
```

```
api_design.md について以下を修正してください。
- moveCard の引数を direction ではなく targetColumnId にしてほしい。
  理由: UIからは「どのカラムに移動するか」が自然なため。
  隣接チェックは moveCard 内部で行う。
```

---

## ステップ4: 成果物を確認する

修正後、生成された詳細設計の最終形を確認する。
今回のチュートリアルでは、レビューと修正を経て、以下のような内容を採用した。

### 成果物: architecture.md

```markdown
# アーキテクチャ

## 技術スタック
- React 18 + TypeScript
- Vite（ビルドツール）
- localStorage（永続化）

## コンポーネント構成

​```
App
├── Board
│   ├── Column（×3: Todo / InProgress / Done）
│   │   ├── ColumnHeader
│   │   └── Card（×N）
│   └── CardCreateForm
└── （将来的に追加される可能性のあるコンポーネント）
​```

## 状態管理
- useState でボード全体の状態を管理
- 状態変更のたびに localStorage に保存

## ディレクトリ構成

​```
implement/
├── src/
│   ├── components/    # UIコンポーネント
│   ├── hooks/         # カスタムフック（状態管理を含む）
│   ├── types/         # 型定義
│   └── storage/       # localStorage操作
├── package.json
├── tsconfig.json
└── vite.config.ts
​```
```

### 成果物: backend/api_design.md

```markdown
# データ構造・インターフェース定義

## 型定義

​```typescript
type ColumnId = 'todo' | 'inProgress' | 'done';

interface Card {
  id: string;
  title: string;
  columnId: ColumnId;
  createdAt: string; // ISO 8601
}

interface Board {
  cards: Card[];
}
​```

## 操作インターフェース

​```typescript
// カード作成
createCard(title: string): Card

// カード移動（隣接カラムのみ。隣接チェックは内部で行う）
moveCard(cardId: string, targetColumnId: ColumnId): void

// カード削除
deleteCard(cardId: string): void

// ボード取得（localStorage から復元）
loadBoard(): Board

// ボード保存（localStorage へ保存）
saveBoard(board: Board): void
​```

## カラムの隣接関係

​```
todo ←→ inProgress ←→ done
​```
```

### 成果物: backend/sequences/card_create.md（例）

```markdown
# シーケンス: カード作成

​```
ユーザー → CardCreateForm: タイトルを入力
CardCreateForm → CardCreateForm: バリデーション（空文字チェック）
  [空文字の場合]
  CardCreateForm → ユーザー: エラーメッセージ表示
  [正常な場合]
  CardCreateForm → useBoard: createCard(title)
  useBoard → Board状態: カードをTodoに追加
  Board状態 → localStorage: 保存
  Board状態 → Board: 再レンダリング
​```
```

### 成果物: test_plan.md

```markdown
# テスト方針

## テスト種別

| 種別 | 対象 | ツール |
|------|------|--------|
| ユニットテスト | 状態管理のロジック | Vitest |
| E2Eテスト | ユーザー操作の全フロー | Playwright |

## カバレッジ方針

### ユニットテスト
- カード作成ロジック（正常系・空文字バリデーション）
- カード移動ロジック（正常系・隣接チェック・飛び越し防止）
- カード削除ロジック
- localStorage の保存・復元

### E2Eテスト
- features/01_card_management.feature の全シナリオ
```

---

## この章の成果物

```
documents/specification/
├── architecture.md       ✅ コンポーネント構成・ディレクトリ構成
├── test_plan.md          ✅ テスト方針
└── backend/
    ├── api_design.md     ✅ データ構造・操作インターフェース
    └── sequences/
        ├── card_create.md    ✅ カード作成フロー
        ├── card_move.md      ✅ カード移動フロー
        └── card_delete.md    ✅ カード削除フロー
```

> **ポイント:** 生成された詳細設計は「人間がレビュー・承認したもの」をコミットする。
> AIの出力をそのまま受け入れるのではなく、判断を経たものが成果物になる。

---

## 振り返り

- **渡す情報を絞る効果:** 要求定義だけを渡すことで、AIは「何を設計すべきか」に集中できた
- **レビューの観点を持つ:** 生成物を「正しいか」ではなく「意図と合っているか」で判断する
- **修正には理由を添える:** 「こう変えて」だけでなく「なぜ変えるか」を伝えると精度が上がる
- **コンテキスト設計の実感:** フェーズを分けて情報を渡す価値が体感できたはず

---

## 次の章へ

第4章では、この詳細設計をAIに渡して実装させる。
featureファイルのシナリオを受け入れ条件として使い、「シナリオを満たしているか」で品質を判断する。
