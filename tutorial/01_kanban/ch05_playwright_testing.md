# 第5章: Playwrightでテストする

## この章のゴール

> **正常系E2Eテストが通る状態**

第4章で手動確認していたシナリオを、Playwrightで自動化する。
featureファイルのシナリオがそのままテストケースになることを体験する。

---

## なぜPlaywrightか

`guide/06_ui_design.md` で述べている通り、UIのテストはGUI・CLIを問わず
バックエンドのGherkinシナリオと **同じGherkin形式** で書ける。
GUIのテスト粒度や設計成果物の詳細は `guide/06_01_gui.md` を参照。

| バックエンド | フロントエンド |
|------------|-------------|
| Gherkinシナリオ | Playwrightシナリオ（同じGherkin） |
| ユニットテスト | コンポーネントテスト |

チュートリアル1では正常系のE2Eテストだけで十分。
Page Objectパターン・APIモック・コンポーネントテストはチュートリアル2以降で導入する。

> `guide/06_01_gui.md` のテスト粒度マトリックス「1人・単機能」の列を参照。

---

## ステップ1: Playwrightをセットアップする

### AIへの依頼

```
implement/ にPlaywrightを導入してください。

- Playwrightのインストールと設定
- テスト用のディレクトリ構成
- npm script にテスト実行コマンドを追加

documents/specification/test_plan.md を参照してください。
```

### 期待される構成

```
implement/
├── e2e/
│   └── card-management.spec.ts   # テストファイル
├── playwright.config.ts           # Playwright設定
└── package.json                   # "test:e2e" スクリプト追加
```

### 人間が確認すること

- `npm run test:e2e` でPlaywrightが起動するか（テストが0件でもエラーにならないか）
- テスト対象のURL（`http://localhost:5173` 等）が正しいか

---

## ステップ2: フロントエンド用のGherkinシナリオを書く

バックエンド用のfeatureファイル（`documents/requirements/features/`）を参考に、
フロントエンド視点のシナリオを作る。

### バックエンドとフロントエンドのシナリオの違い

| 観点 | バックエンド（第2章で作成済み） | フロントエンド（この章で作成） |
|------|------|------|
| 視点 | データ・ロジックの振る舞い | ユーザーが画面で見る振る舞い |
| 操作 | 関数呼び出し・APIリクエスト | ボタンクリック・フォーム入力 |
| 検証 | 戻り値・状態変更 | 画面表示・要素の有無 |

> このチュートリアルではバックエンドがないため差が小さいが、
> チュートリアル2以降でバックエンドが入ると区別が重要になる。

### AIへの依頼

```
documents/requirements/features/01_card_management.feature を読んで、
フロントエンドE2Eテスト用のGherkinシナリオを書いてください。

documents/requirements/frontend/features/ に配置してください。
ユーザーの画面操作の視点で書いてください。

まずは正常系のシナリオだけで構いません。
```

### 成果物のイメージ: frontend/features/01_card_management.feature

```gherkin
Feature: カード管理（E2E）

  Kanbanボード上でのカード操作をブラウザで検証する。

  Background:
    Given ブラウザでKanbanボードを開いている

  # --- 作成 ---

  Scenario: カードを新規作成する
    When タイトル入力欄に「買い物リストを作る」と入力する
    And 「作成」ボタンをクリックする
    Then Todoカラムに「買い物リストを作る」カードが表示される

  # --- 移動 ---

  Scenario: カードをTodoからIn Progressに移動する
    Given Todoカラムに「タスクA」カードが存在する
    When 「タスクA」カードの「→」ボタンをクリックする
    Then In Progressカラムに「タスクA」カードが表示される
    And Todoカラムに「タスクA」カードが表示されていない

  Scenario: カードをIn ProgressからDoneに移動する
    Given In Progressカラムに「タスクA」カードが存在する
    When 「タスクA」カードの「→」ボタンをクリックする
    Then Doneカラムに「タスクA」カードが表示される

  # --- 削除 ---

  Scenario: カードを削除する
    Given Todoカラムに「タスクA」カードが存在する
    When 「タスクA」カードの「×」ボタンをクリックする
    Then ボードに「タスクA」カードが表示されていない

  # --- 永続化 ---

  Scenario: リロード後もカードが保持される
    Given Todoカラムに「タスクA」カードが存在する
    When ページをリロードする
    Then Todoカラムに「タスクA」カードが表示される
```

### 人間がレビューすること

- バックエンドのfeatureファイルの正常系シナリオがカバーされているか
- 操作が具体的か（「ボタンをクリックする」等、UIの操作が明記されているか）
- 検証が画面表示の視点になっているか

---

## ステップ3: Playwrightテストを実装する

### AIへの依頼

```
documents/requirements/frontend/features/01_card_management.feature の
シナリオをPlaywrightテストとして実装してください。

implement/e2e/card-management.spec.ts に書いてください。

テストの前提:
- dev サーバーが起動済みであること
- テスト開始時に localStorage をクリアすること（テスト間の独立性）
```

### 成果物のイメージ: e2e/card-management.spec.ts

```typescript
import { test, expect } from '@playwright/test';

test.beforeEach(async ({ page }) => {
  // localStorage をクリアしてテスト間の独立性を確保
  await page.goto('/');
  await page.evaluate(() => localStorage.clear());
  await page.reload();
});

test.describe('カード作成', () => {
  test('カードを新規作成する', async ({ page }) => {
    await page.goto('/');

    await page.getByPlaceholder('タイトルを入力').fill('買い物リストを作る');
    await page.getByRole('button', { name: '作成' }).click();

    const todoColumn = page.locator('[data-column="todo"]');
    await expect(todoColumn).toContainText('買い物リストを作る');
  });
});

test.describe('カード移動', () => {
  test('カードをTodoからIn Progressに移動する', async ({ page }) => {
    await page.goto('/');

    // カードを作成
    await page.getByPlaceholder('タイトルを入力').fill('タスクA');
    await page.getByRole('button', { name: '作成' }).click();

    // 移動
    const card = page.locator('text=タスクA');
    await card.locator('..').getByRole('button', { name: '→' }).click();

    // 検証
    const inProgressColumn = page.locator('[data-column="inProgress"]');
    const todoColumn = page.locator('[data-column="todo"]');
    await expect(inProgressColumn).toContainText('タスクA');
    await expect(todoColumn).not.toContainText('タスクA');
  });
});

// ... 他のテストも同様
```

> **注意:** 上のコードはイメージであり、実際のセレクターは実装に依存する。
> AIは実装済みのコードを読んで適切なセレクターを使ってくれる。

### 人間が確認すること

- テストが実際に通るか（`npm run test:e2e`）
- テスト間が独立しているか（localStorage のクリア）
- セレクターが実装と一致しているか

---

## ステップ4: テストを実行する

### 実行コマンド

```bash
# dev サーバーを起動（別ターミナル）
cd implement && npm run dev

# テスト実行
cd implement && npm run test:e2e
```

または、Playwrightの `webServer` 設定を使えばサーバー起動を自動化できる。

### AIへの依頼（webServer設定の追加）

```
playwright.config.ts に webServer 設定を追加して、
テスト実行時に自動で dev サーバーが起動するようにしてください。
```

### テスト結果の確認

全テストがパスすれば、この章のゴール達成。

```
Running 5 tests using 1 worker

  ✓ カード作成 > カードを新規作成する
  ✓ カード移動 > カードをTodoからIn Progressに移動する
  ✓ カード移動 > カードをIn ProgressからDoneに移動する
  ✓ カード削除 > カードを削除する
  ✓ 永続化 > リロード後もカードが保持される

  5 passed
```

### テストが失敗したとき

```
テスト「カードをTodoからIn Progressに移動する」が失敗しました。
以下がエラーです。

[エラーメッセージを貼る]

実装を修正してテストが通るようにしてください。
```

テストが実装の受け入れ条件として機能していることを実感できるはずだ。

---

## この章の成果物

```
documents/requirements/frontend/
└── features/
    └── 01_card_management.feature  ✅ フロントエンドE2Eシナリオ

implement/
├── e2e/
│   └── card-management.spec.ts    ✅ Playwrightテスト
└── playwright.config.ts            ✅ Playwright設定
```

正常系E2Eテストが全件パスしている状態。

---

## 振り返り

- **featureファイルからテストへ:** Gherkinシナリオがそのままテストケースの設計図になった
- **バックエンドとフロントエンドの対称性:** 同じGherkin形式で書けることを体験した
- **自動テストの安心感:** 手動確認を自動化したことで、今後の変更時にリグレッションを検出できる
- **テストの粒度:** チュートリアル1では正常系E2Eだけで十分。異常系・Page Object・APIモックはチュートリアル2で

---

## 次の章へ

次は振り返りの章で、チュートリアル1全体を通じて何を学んだかを整理する。
