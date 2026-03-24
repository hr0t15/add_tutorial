# 第5章: Page Object・APIモックを導入する

## この章のゴール

> **保守性の高いE2Eテストが通る状態**

チュートリアル1ではシンプルなPlaywrightテストを書いた。
このチュートリアルでは **Page Objectパターン** と **APIモック** を導入し、
テストの保守性と安定性を上げる。

---

## なぜPage ObjectとAPIモックが必要か

### チュートリアル1のテストの問題点

```typescript
// チュートリアル1のスタイル
test('カードを作成する', async ({ page }) => {
  await page.getByPlaceholder('タイトルを入力').fill('タスクA');
  await page.getByRole('button', { name: '作成' }).click();
  await expect(page.locator('[data-column="todo"]')).toContainText('タスクA');
});
```

このスタイルは小規模では問題ないが、画面が増えると以下の問題が出る。

| 問題 | 説明 |
|------|------|
| **セレクターの重複** | 同じセレクターが複数テストに散らばる。UIを変更すると全テストを修正する必要がある |
| **テストの意図が読みにくい** | セレクターの詳細がノイズになり、「何をテストしているか」が見えにくい |
| **バックエンド依存** | バックエンドが動いていないとテストが全滅する |

### 解決策

```
Page Object     → セレクターを1箇所にまとめ、テストからUIの詳細を隠す
APIモック        → バックエンドの応答を差し替え、フロントエンド単独でテスト可能にする
```

> `guide/06_01_gui.md` のテスト粒度マトリックス「小チーム・複数機能」の列を参照。
> この列で Page Object が「○ 推奨」、APIモックが「○ 推奨」になっているのは、
> まさにこのチュートリアルの規模（複数アクター・複数画面）がその閾値に達しているためである。
> チュートリアル1（1人・単機能）では不要だったこれらのパターンが、
> 規模が上がることで必要になる過程を体感してほしい。

---

## ステップ1: Page Objectパターンを導入する

### Page Objectとは

画面（ページ）の操作をクラスに閉じ込めるパターン。
テストコードはPage Objectのメソッドを呼ぶだけになる。

```
変更前: テストがセレクターを直接操作する
  test → page.locator('[data-column="todo"]') → UI

変更後: Page Objectがセレクターを隠蔽する
  test → orderPage.placeOrder('T1', ['カレー']) → UI
```

### AIへの依頼

```
Playwrightテストに Page Objectパターンを導入してください。

以下の画面ごとにPage Objectクラスを作ってください。
- TableListPage（テーブル一覧画面）
- OrderPage（注文画面）
- KitchenPage（厨房画面）
- ServingPage（配膳画面）

各Page Objectには以下を含めてください。
- 画面遷移メソッド（goto）
- 操作メソッド（ボタンクリック等）
- アサーションメソッド（表示確認等）
```

### 成果物のイメージ

```
implement/
└── e2e/
    ├── pages/
    │   ├── table-list.page.ts
    │   ├── order.page.ts
    │   ├── kitchen.page.ts
    │   └── serving.page.ts
    ├── order.spec.ts
    ├── kitchen.spec.ts
    ├── serving.spec.ts
    └── billing.spec.ts
```

### Page Objectの例: order.page.ts

```typescript
import { Page, expect } from '@playwright/test';

export class OrderPage {
  constructor(private page: Page) {}

  async goto(tableId: string) {
    await this.page.goto(`/tables/${tableId}/order`);
  }

  async addItem(itemName: string) {
    await this.page.getByRole('button', { name: itemName }).click();
  }

  async submitOrder() {
    await this.page.getByRole('button', { name: '注文する' }).click();
  }

  async cancelItem(itemName: string) {
    const item = this.page.locator(`[data-item-name="${itemName}"]`);
    await item.getByRole('button', { name: 'キャンセル' }).click();
  }

  async expectItemInOrder(itemName: string) {
    await expect(this.page.locator('.order-items')).toContainText(itemName);
  }

  async expectError(message: string) {
    await expect(this.page.locator('.error-message')).toContainText(message);
  }
}
```

### Page Objectを使ったテスト

```typescript
import { test } from '@playwright/test';
import { OrderPage } from './pages/order.page';
import { KitchenPage } from './pages/kitchen.page';

test('テーブルの注文を入力する', async ({ page }) => {
  const orderPage = new OrderPage(page);
  const kitchenPage = new KitchenPage(page);

  await orderPage.goto('T1');
  await orderPage.addItem('カレー');
  await orderPage.addItem('サラダ');
  await orderPage.submitOrder();

  await orderPage.expectItemInOrder('カレー');
  await orderPage.expectItemInOrder('サラダ');

  await kitchenPage.goto();
  await kitchenPage.expectItem('T1', 'カレー');
  await kitchenPage.expectItem('T1', 'サラダ');
});
```

> **比較:** セレクターの詳細が消え、テストが「何をしているか」が読み取りやすくなった。

### 人間が確認すること

- Page Objectのメソッド名がユビキタス言語と一致しているか
- テストコードからセレクターが消えているか
- UIを変更したときにPage Objectだけ修正すれば済む構造になっているか

---

## ステップ2: APIモックを導入する

### なぜAPIモックが必要か

レストランシステムにはバックエンドがある（ADR-001で決定）。
テスト実行時にバックエンドが動いていないと、フロントエンドのテストが全滅する。

APIモックを使うと:
- バックエンドなしでフロントエンドだけテストできる
- バックエンドの応答を自由にコントロールできる（エラー応答も作れる）
- テストが高速になる（ネットワーク通信がない）

### AIへの依頼

```
PlaywrightのAPIモック機能（page.route）を使って、
バックエンドなしでフロントエンドをテストできるようにしてください。

以下のAPIをモックしてください。
- GET /api/menu（メニュー取得）
- POST /api/tables/:tableId/orders（注文作成）
- GET /api/kitchen/items（厨房品目取得）
- PATCH /api/order-items/:itemId/cook（調理完了）

モックデータは fixtures/ ディレクトリにまとめてください。
```

### 成果物の構成

```
implement/
└── e2e/
    ├── fixtures/
    │   ├── menu.json              # メニューデータ
    │   ├── order-response.json    # 注文作成レスポンス
    │   └── kitchen-items.json     # 厨房品目データ
    ├── helpers/
    │   └── api-mock.ts            # APIモック設定
    ├── pages/
    │   └── ...
    └── ...
```

### APIモックの例: api-mock.ts

```typescript
import { Page } from '@playwright/test';
import menu from '../fixtures/menu.json';

export async function setupApiMock(page: Page) {
  // メニュー取得
  await page.route('**/api/menu', (route) => {
    route.fulfill({ json: menu });
  });

  // 注文作成
  await page.route('**/api/tables/*/orders', (route) => {
    if (route.request().method() === 'POST') {
      route.fulfill({
        json: { id: 'order-1', tableId: 'T1', items: [] },
      });
    }
  });
}

export async function mockUnavailableItem(page: Page, itemName: string) {
  // 品切れ状態のメニューを返す
  const unavailableMenu = menu.map((item) =>
    item.name === itemName ? { ...item, available: false } : item
  );
  await page.route('**/api/menu', (route) => {
    route.fulfill({ json: unavailableMenu });
  });
}
```

### APIモックを使ったテスト

```typescript
import { test } from '@playwright/test';
import { OrderPage } from './pages/order.page';
import { setupApiMock, mockUnavailableItem } from './helpers/api-mock';

test('品切れの品目を注文しようとする', async ({ page }) => {
  await setupApiMock(page);
  await mockUnavailableItem(page, 'カレー');

  const orderPage = new OrderPage(page);
  await orderPage.goto('T1');
  await orderPage.addItem('カレー');

  await orderPage.expectError('品切れです');
});
```

> **ポイント:** `mockUnavailableItem` で「品切れ」の状態を作れる。
> バックエンドが動いていなくても異常系テストが書ける。

---

## ステップ3: テストを整理して実行する

### テストファイルの構成

featureファイルと対応する形でテストファイルを整理する。

| featureファイル | テストファイル | Page Object |
|---------------|-------------|-------------|
| 01_order.feature | order.spec.ts | OrderPage |
| 02_kitchen.feature | kitchen.spec.ts | KitchenPage |
| 03_serving.feature | serving.spec.ts | ServingPage |
| 04_billing.feature | billing.spec.ts | TableListPage |

### テスト実行

```bash
cd implement && npm run test:e2e
```

### 期待される結果

```
Running 10 tests using 1 worker

  ✓ 注文 > テーブルの注文を入力する
  ✓ 注文 > 追加注文をする
  ✓ 注文 > 厨房受付前に注文品目をキャンセルする
  ✓ 注文 > 品切れの品目を注文しようとする
  ✓ 注文 > 調理完了した品目はキャンセルできない
  ✓ 厨房 > 厨房が調理完了を記録する
  ✓ 配膳 > サーバーが配膳済みにする
  ✓ 配膳 > 調理未完了の品目は配膳できない
  ✓ 会計 > テーブルの会計を行う
  ✓ 会計 > 未配膳の品目がある状態で会計する

  10 passed
```

---

## この章の成果物

```
implement/e2e/
├── fixtures/                ✅ APIモックデータ
├── helpers/
│   └── api-mock.ts          ✅ APIモック設定
├── pages/
│   ├── table-list.page.ts   ✅ Page Object
│   ├── order.page.ts        ✅ Page Object
│   ├── kitchen.page.ts      ✅ Page Object
│   └── serving.page.ts      ✅ Page Object
├── order.spec.ts            ✅ 注文テスト（5本）
├── kitchen.spec.ts          ✅ 厨房テスト（1本）
├── serving.spec.ts          ✅ 配膳テスト（2本）
└── billing.spec.ts          ✅ 会計テスト（2本）
```

---

## 振り返り

- **Page Objectの価値:** テストコードが「何をしているか」に集中できるようになった。
  UIの変更があってもPage Objectだけ修正すればよい
- **APIモックの価値:** バックエンドなしで異常系テストが書ける。
  「品切れ」「調理完了後のキャンセル」などの状態を自由に作れる
- **テストとfeatureファイルの対応:** featureファイルの構造がそのままテストの構造になった

---

## 次の章へ

振り返りの章で、チュートリアル2全体を通じて
「イベントストーミングの前後で何が変わったか」を整理する。
