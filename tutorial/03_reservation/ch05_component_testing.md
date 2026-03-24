# 第5章: コンポーネントテストを追加する

## この章のゴール

> **ドメインルールがコンポーネントテストで保護されている状態**

チュートリアル1ではE2Eテストのみ、チュートリアル2ではE2E + Page Object + APIモックを導入した。
このチュートリアルでは **コンポーネントテスト** を新たに導入する。

---

## なぜコンポーネントテストが必要か

### E2Eテストだけでは不十分なケース

二重予約防止を例にとる。

```
E2Eテスト:
  「ブラウザでスタッフAの10:00を予約 → 別のブラウザで同じ枠を予約しようとする
   → エラーが表示される」

  → 正しい。だが、テストの実行に時間がかかり、失敗原因の特定が困難。
  → ブラウザ2つを同時に操作するテストは不安定になりやすい。
```

```
コンポーネントテスト:
  「TimeSlot集約に対してreserve()を2回呼ぶ
   → 2回目は例外が発生する」

  → ドメインルールを直接テストしている。高速で安定。
  → 失敗時に「何が壊れたか」が一目瞭然。
```

### テストピラミッド

```
        △ E2E テスト
       ／ ＼  （少数・遅い・ユーザー操作の確認）
      ／───＼
     ／      ＼  コンポーネントテスト ← NEW
    ／────────＼  （中程度・ドメインルールの確認）
   ／          ＼
  ／────────────＼ ユニットテスト
  ─────────────── （多数・速い・個別関数の確認）
```

コンポーネントテストは、ドメインルール（集約・ポリシー）を
UIを経由せずにテストするレイヤー。

> `guide/06_01_gui.md` のテスト粒度マトリックスでは、
> コンポーネントテストは「中規模・複数チーム」から推奨されている。

---

## テスト対象の整理

### 何をコンポーネントテストで書くか

| ドメインルール | テスト内容 | E2Eでカバー？ |
|-------------|----------|-------------|
| 二重予約防止 | 同一枠に2つ目の予約が失敗する | △ 不安定 |
| 同一顧客の重複予約防止 | 同一顧客が同じ時間帯に2つ予約できない | △ 不安定 |
| キャンセル待ち昇格 | キャンセル時に待機者が自動予約される | ○ 可能だが遅い |
| 昇格のFIFO順 | 先に登録した人が先に昇格する | ○ 可能だが遅い |
| 楽観的ロック | バージョン競合時にエラーになる | × テスト困難 |
| 状態遷移の制約 | 空き→予約済みのみ、逆は不可 | ○ 可能 |

> **基準:** 「E2Eでは不安定またはテスト困難」かつ「ビジネス上重要」なルールを
> コンポーネントテストで書く。

---

## ステップ1: 二重予約防止のテスト

### AIへの依頼

```
TimeSlot 集約の二重予約防止ルールをテストするコンポーネントテストを書いてください。

テスト対象:
- TimeSlot.reserve() が予約済みの枠に対して呼ばれたら例外を投げる
- データベースを使い、楽観的ロックが正しく動作することを確認する

以下のシナリオに対応するテストを書いてください。

Scenario: 同一スタッフの同一時間帯に二重予約できない
  Given スタッフAの10:00に「山田太郎」の予約がある
  When 顧客「佐藤花子」がスタッフAの10:00に予約しようとする
  Then 予約は作成されない
```

### テストのイメージ

```typescript
describe('TimeSlot 集約 - 二重予約防止', () => {
  test('予約済みの枠に二重予約できない', async () => {
    // Given: スタッフAの10:00枠を作成し、山田太郎が予約
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');
    await timeSlot.reserve('customer-yamada', 'menu-cut');

    // When & Then: 佐藤花子が同じ枠に予約しようとする → 例外
    await expect(
      timeSlot.reserve('customer-sato', 'menu-cut')
    ).rejects.toThrow('この枠はすでに予約済みです');
  });

  test('楽観的ロックによる競合検出', async () => {
    // Given: 同じTimeSlotを2つのトランザクションが同時に読み込む
    const timeSlot1 = await loadTimeSlot('staffA', '2024-01-15T10:00');
    const timeSlot2 = await loadTimeSlot('staffA', '2024-01-15T10:00');

    // When: 1つ目が予約に成功
    await timeSlot1.reserve('customer-yamada', 'menu-cut');
    await saveTimeSlot(timeSlot1);

    // Then: 2つ目はバージョン競合で失敗
    timeSlot2.reserve('customer-sato', 'menu-cut');
    await expect(
      saveTimeSlot(timeSlot2)
    ).rejects.toThrow('競合が検出されました');
  });
});
```

> **ポイント:** 2つ目のテストは楽観的ロックのテスト。
> E2Eでは「2つのブラウザを同時に操作する」必要があり非常に不安定だが、
> コンポーネントテストでは「2つのオブジェクトを同時に操作する」だけで済む。

---

## ステップ2: キャンセル待ち昇格のテスト

### AIへの依頼

```
キャンセル待ち昇格ポリシーのコンポーネントテストを書いてください。

テスト対象:
- キャンセル時にキャンセル待ちの先頭が自動予約される
- FIFO順が正しく守られる
- キャンセル待ちがいない場合は枠が空きに戻る
```

### テストのイメージ

```typescript
describe('キャンセル待ち昇格ポリシー', () => {
  test('キャンセル時に待機者の先頭が自動昇格する', async () => {
    // Given: 予約あり + キャンセル待ち2人
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');
    await timeSlot.reserve('customer-yamada', 'menu-cut');
    await timeSlot.joinWaitingList('customer-sato');
    await timeSlot.joinWaitingList('customer-suzuki');

    // When: 予約をキャンセル
    await timeSlot.cancel();

    // Then: 佐藤花子が自動昇格
    expect(timeSlot.reservation?.customerId).toBe('customer-sato');
    expect(timeSlot.waitingList).toHaveLength(1);
    expect(timeSlot.waitingList[0].customerId).toBe('customer-suzuki');
  });

  test('キャンセル待ちがいない場合は空きに戻る', async () => {
    // Given: 予約あり、キャンセル待ちなし
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');
    await timeSlot.reserve('customer-yamada', 'menu-cut');

    // When: 予約をキャンセル
    await timeSlot.cancel();

    // Then: 空きに戻る
    expect(timeSlot.reservation).toBeNull();
    expect(timeSlot.isAvailable()).toBe(true);
  });

  test('FIFO順が守られる', async () => {
    // Given: 佐藤 → 鈴木の順でキャンセル待ち
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');
    await timeSlot.reserve('customer-yamada', 'menu-cut');
    await timeSlot.joinWaitingList('customer-sato');    // 先
    await timeSlot.joinWaitingList('customer-suzuki');   // 後

    // When: キャンセル
    await timeSlot.cancel();

    // Then: 佐藤が先に昇格（FIFO）
    expect(timeSlot.reservation?.customerId).toBe('customer-sato');
  });
});
```

---

## ステップ3: 状態遷移の制約テスト

### AIへの依頼

```
TimeSlot の状態遷移が正しく制約されていることを
テストするコンポーネントテストを書いてください。

domain_model.md の状態遷移図を参照してください。

​```
available → reserved  [reserve() 成功時]
reserved → available  [cancel() かつ キャンセル待ちなし]
reserved → reserved   [cancel() かつ キャンセル待ちあり]
​```
```

### テストのイメージ

```typescript
describe('TimeSlot 状態遷移', () => {
  test('空き枠に予約できる', async () => {
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');
    expect(timeSlot.isAvailable()).toBe(true);

    await timeSlot.reserve('customer-yamada', 'menu-cut');
    expect(timeSlot.isAvailable()).toBe(false);
  });

  test('空き枠にキャンセルは不可', async () => {
    const timeSlot = await createTimeSlot('staffA', '2024-01-15T10:00');

    await expect(timeSlot.cancel()).rejects.toThrow('予約がありません');
  });
});
```

---

## テストの実行

### コマンド

```bash
# コンポーネントテスト
cd implement && npm run test:component

# E2Eテスト（チュートリアル2の手法を引き続き使用）
cd implement && npm run test:e2e

# 全テスト
cd implement && npm run test
```

### 期待される結果

```
Component Tests:
  TimeSlot 集約 - 二重予約防止
    ✓ 予約済みの枠に二重予約できない
    ✓ 楽観的ロックによる競合検出
  キャンセル待ち昇格ポリシー
    ✓ キャンセル時に待機者の先頭が自動昇格する
    ✓ キャンセル待ちがいない場合は空きに戻る
    ✓ FIFO順が守られる
  TimeSlot 状態遷移
    ✓ 空き枠に予約できる
    ✓ 空き枠にキャンセルは不可

  7 passed (0.3s)

E2E Tests:
  （第4章で確認済みのシナリオ）
  8 passed (15s)
```

> コンポーネントテストは0.3秒、E2Eテストは15秒。
> ドメインルールの検証はコンポーネントテストの方が圧倒的に速い。

---

## E2Eテストとコンポーネントテストの使い分け

| テスト種別 | テスト対象 | 速度 | 安定性 | 確認できること |
|-----------|----------|------|--------|-------------|
| **E2E** | ユーザー操作の全フロー | 遅い | △ | 画面遷移・表示・操作 |
| **コンポーネント** | ドメインルール | 速い | ◎ | ビジネスロジックの正しさ |

```
方針:
  ドメインルール（二重予約防止、昇格ポリシー、状態遷移）
    → コンポーネントテストで保護する

  ユーザー操作（予約画面で枠をクリック → 予約作成 → 表示確認）
    → E2Eテストで確認する

  両方でカバーする必要はない。
  「このルールが壊れたとき、どのテストが検知するか」が1つ以上あればよい。
```

---

## この章の成果物

```
implement/
└── tests/
    ├── component/
    │   ├── time-slot-reservation.test.ts   ✅ 二重予約防止（2本）
    │   ├── waiting-list-promotion.test.ts  ✅ キャンセル待ち昇格（3本）
    │   └── time-slot-state.test.ts         ✅ 状態遷移（2本）
    └── e2e/
        └── （第4章で作成済み）
```

コンポーネントテスト 7本 + E2Eテスト 8本 = 合計15本がすべてパス。

---

## 振り返り

- **コンポーネントテストの価値:** ドメインルール（二重予約防止・キャンセル待ち昇格）を
  高速・安定にテストできるようになった
- **テストの使い分け:** E2Eは「ユーザー操作が正しく動くか」、
  コンポーネントは「ビジネスロジックが正しいか」と役割を分けた
- **集約がテストを簡潔にする:** TimeSlot集約を直接テストすることで、
  UIやAPIを経由せずにドメインルールを検証できた
- **楽観的ロックのテスト:** E2Eでは困難な競合テストが、
  コンポーネントテストでは自然に書けた

---

## 次の章へ

振り返りの章で、3つのチュートリアル全体を通じて何を学んだかを整理し、
実務への応用ガイドを提供する。
