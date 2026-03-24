# Appendix I: ハーネスエンジニアリング

> **ハーネスエンジニアリング**とは、コーディングAIが動作する**実行環境**を設計・改善する技術である。
> AIモデルそのものではなく、モデルの**外側**——指示ファイル・リンター・Hooks・CI・検証ループ——を
> 設計対象とする。
>
> 2026年に入り急速に重要性が高まっている領域であるが、
> モデル能力の向上やツール側の標準化によって**将来的には不要になる部分も多い**と予想される。
> このAppendixは「2026年時点のベストプラクティス仮説」として読んでほしい。

---

## ハーネスエンジニアリングとは何か

### 定義

```
Agent = Model + Harness
```

エージェントの出力品質は、モデルの能力だけでなく**ハーネス（実行環境）の設計**によって大きく変わる。
LangChainはハーネスの改善だけでTerminal Bench 2.0のスコアを52.8%から66.5%に向上させた（モデル変更なし）。

### 概念の進化

```
プロンプトエンジニアリング（2023）
  → 1回の指示を工夫する

コンテキストエンジニアリング（2024-2025）
  → AIに渡す情報をシステムとして設計する

ハーネスエンジニアリング（2026〜）
  → AIが動作する実行環境全体を設計する
```

### Fowlerの3カテゴリ分類

Martin Fowler / Birgitta Bockeler（Thoughtworks）は、ハーネスを以下の3つに分類している。

| カテゴリ | 内容 | 本ガイドでの対応 |
|---------|------|-----------------|
| **コンテキストエンジニアリング** | AIに渡す情報の設計 | `appendix/F_context_engineering.md` |
| **アーキテクチャ制約** | リンター・構造テスト・Hooks | **このAppendixで扱う** |
| **メンテナンスエージェント** | 自動品質維持（ガベージコレクション的） | **このAppendixで扱う** |

コンテキストエンジニアリングはハーネスの一部であるが、本ガイドでは関心の軸と寿命が異なるため別のAppendixとして扱っている。

---

## このガイドとの関係

### 上流と下流

```
本ガイドが扱う領域（上流）:
  要求定義 → 詳細設計
  = 「何を・どう作るか」を構造化する
  = コンテキストエンジニアリング側

ハーネスエンジニアリングが扱う領域（下流）:
  実装時の実行環境
  = 「AIがどう動く環境を作るか」を設計する
  = アーキテクチャ制約 + メンテナンスエージェント側
```

### 重なる部分

本ガイドの成果物のうち、ハーネスエンジニアリングの観点でも価値が高いものがある。

| 成果物 | ハーネスとしての価値 |
|--------|-------------------|
| **Gherkinシナリオ** | 受け入れ条件 兼 自動テスト。エージェントの自己検証に使える |
| **ADR** | 判断の不可変な記録。エージェントが「なぜこうなっているか」を参照できる |
| **ユビキタス言語** | CLAUDE.md/AGENTS.mdに用語ポインタとして含められる |

### 緊張関係:「ドキュメントは腐る」問題

ハーネスエンジニアリングの文献では「記述的ドキュメントは腐る。テストは腐らない」という主張がある。
エージェントはリポジトリ内の全テキストを等しく権威あるものとして扱うため、
古い記述が残っていると誤った方向に実装する。

本ガイドはoverviewmd・domain_model.md等の記述的ドキュメントを成果物として定義しているが、
これらには以下の位置づけを与える。

```
記述的成果物（overview.md, domain_model.md）:
  → 発見プロセス（フェーズ1）での思考整理に不可欠
  → フェーズ完了後は「シナリオとテスト」に変換されるべき
  → コーディングAIには直接渡さず、シナリオ経由で伝える

実行可能な成果物（Gherkinシナリオ, テスト, ADR）:
  → 腐りにくい。ハーネスと相性が良い
  → コーディングAIに直接渡してよい
```

> この使い分けは `appendix/F_context_engineering.md` の「ポインタ渡しパターン」とも整合する。
> フェーズ3でAIに渡すのは「結論（シナリオ・設計書）」であり、「過程（壁打ちメモ・概要書）」ではない。

---

## 核心原則

### 1. 決定論的ツールで品質を保証する

**「プロンプトは提案、Hooksは保証」**（Anthropic）

AIへの指示は確率的であり、従わないことがある。
リンター・フォーマッター・Hooksは決定論的であり、**必ず**実行される。

```
フィードバック速度の階層:

  PostToolUse Hooks (ms)     ← 最速。ファイル保存のたびに実行
    ↓
  pre-commit hooks (秒)      ← コミット前に自動実行
    ↓
  CI/CD (分)                 ← プッシュ後に実行
    ↓
  人間のレビュー (時間)       ← 最も遅い

→ 上位で検出できるほど修正コストが低い
→ ハーネス設計の最大のレバレッジはここにある
```

#### 言語別推奨ツール

| 言語 | リンター | フォーマッター | 備考 |
|------|---------|-------------|------|
| TypeScript | Oxlint → ESLint | Biome / Prettier | Oxlintは高速だがルール数が少ない |
| Python | Ruff | Ruff | リンターとフォーマッターを兼ねる |
| Go | golangci-lint | gofmt / goimports | 標準ツールが強力 |
| Rust | Clippy | rustfmt | cargo clippy で統合 |

> **原則:** リンターやフォーマッターの仕事をLLMに委ねない。
> LLMは「正しいコードを書く」ために使い、「整形する」のは決定論的ツールに任せる。

### 2. エージェント指示ファイルはポインタとして書く

CLAUDE.md / AGENTS.md は**散文ではなくポインタ**として設計する。

```
目標: 50行以下
上限: 200行

含めるべきもの:
  - ビルド・テストの実行コマンド
  - コードスタイルの簡潔なルール
  - 成果物へのファイルパスポインタ
  - 「やってはいけないこと」リスト

含めるべきでないもの:
  - コードを読めば分かること
  - 長い説明文や背景
  - LLMが自動生成した内容（逆効果: ETH Zurich研究）
```

#### 本ガイドの成果物との組み合わせ例

```markdown
# CLAUDE.md の例

## プロジェクト構造
- 要求定義: documents/requirements/ を参照
- 詳細設計: documents/specification/ を参照
- シナリオ: documents/requirements/features/*.feature を参照

## ビルド・テスト
- `npm run build` でビルド
- `npm run test` で全テスト実行
- `npx playwright test` でE2Eテスト

## コードスタイル
- TypeScript strict mode
- 関数名は camelCase、型名は PascalCase

## やってはいけないこと
- .env ファイルをコミットしない
- テストなしでPRを出さない
- 既存のADRの判断を覆す変更をしない（新しいADRを書くこと）
```

### 3. 計画と実行を分離する

本ガイドの3フェーズ（要求定義 → 詳細設計 → 実装）は、
ハーネスエンジニアリングの「計画と実行の分離」原則と一致する。

```
本ガイドの3フェーズ:
  フェーズ1（要求定義）→ フェーズ2（詳細設計）→ フェーズ3（実装）
                                                    ↑
                                              ここでコーディングAIに委譲

Anthropicの2エージェントパターン:
  Initializer Agent → Coding Agent
  （環境準備・計画）    （実装）
```

セッション間の状態管理も重要である。

- **Gitをセッション間のブリッジとして使う:** コミットが「ここまで完了」の記録になる
- **進捗管理にはJSONを使う:** Markdownよりもモデルが壊しにくい
- **起動時に前回の状態を検証する:** 前提が変わっていないか確認してから作業を開始する

### 4. 自己検証ループを組み込む

エージェントが自分の出力を検証できる仕組みを設計する。

#### Spotifyの多段検証パイプライン

```
段階1: 決定論的検証
  → フォーマット・ビルド・テストの実行（全自動）

段階2: 出力解析
  → テスト結果を正規表現でパース（全自動）

段階3: LLMジャッジ
  → 変更がプロンプトの意図と合致しているか評価
  → 約25%のセッションを却下、うち50%は自動修正に成功

段階4: 人間レビュー
  → 最終承認
```

Spotifyはこのパイプラインで1,500以上のPRを本番マージしている。

#### Stripeの3段階フィードバック

```
L1: ローカルリンター（5秒以内）
L2: 選択的CI（必要なテストだけ実行）
L3: 修正は2回まで（2回失敗したら人間にエスカレーション）
```

Stripeは週1,300以上のPRを完全自動マージしている。

> **ポイント:** 最も危険な失敗は「CIを通過するが機能的に誤っている」ケース。
> これが信頼を最も損なう。LLMジャッジや人間レビューで対処する層が必要。

---

## ツール別ハーネス構成

### Claude Code

```
コンテキスト設定:
  CLAUDE.md           → プロジェクトルート（チームで共有）
  .claude/settings.json → パーミッション・拒否ルール
  Skills              → ドメイン知識の段階的開示

決定論的制御:
  Hooks:
    PreToolUse        → 破壊的操作の防止
    PostToolUse       → フォーマッター自動実行、テスト実行
    Stop              → 最終検証（LLMジャッジ等）
    SessionStart      → 環境検証・前回の状態確認

Sub-agents           → コンテキストファイアウォール（独立した作業単位）
```

### Codex

```
コンテキスト設定:
  AGENTS.md           → 階層化された指示ファイル
  サンドボックス       → 隔離環境での自動実行

特徴:
  「サンドボックスモデル」— 隔離環境で自律実行し、結果をPRとして提出
  並列実行に強い（複数タスクを同時に処理）
```

### Gemini CLI

```
コンテキスト設定:
  GEMINI.md           → コンテキストファイル階層
  Policy Engine       → ツール実行ルール（writeツールはデフォルトask_user）
  Extensions          → モジュラーなカスタマイズ

Custom Sub-agents    → 独立したコンテキストウィンドウとシステムプロンプト
```

### Cursor

```
コンテキスト設定:
  .cursor/rules/*.mdc → モジュラーなルールファイル（必要なルールだけ活性化）
  .cursorrules        → 非推奨（.mdc形式へ移行）

特徴:
  .mdcファイルの分割によりトークン使用量を削減
  IDE統合による視覚的フィードバック
```

---

## Minimum Viable Harness（MVH）ロードマップ

すべてを一度に導入する必要はない。以下の段階で始める。

### Week 1: 最小構成

```
✅ リンター・フォーマッターの導入（言語に応じて選択）
✅ CLAUDE.md / AGENTS.md を10行で書く（ビルド・テストコマンドのみ）
✅ .gitignore で秘密情報を除外
```

### Week 2-4: フィードバックループの追加

```
✅ pre-commit hooks（フォーマット・リント・テスト）
✅ Hooks設定（PostToolUse: フォーマッター、PreToolUse: 破壊的操作防止）
✅ CLAUDE.md / AGENTS.md を30-50行に拡充（コードスタイル・禁止事項）
```

### Month 2+: 検証の高度化

```
✅ CI統合（PR作成時に自動テスト）
✅ 構造テスト（アーキテクチャの不変量を検証）
✅ LLMジャッジ（変更がプロンプトの意図と合致しているか評価）
✅ ADR参照ルール（既存の判断を覆す変更を検出）
```

> **原則:** エージェントがミスしたら、同じミスを二度と起こさないよう環境を改善する。
> CLAUDE.mdやHooksの各行は、過去の失敗への対策に対応しているのが理想。
> （Mitchell Hashimoto）

---

## この領域の成熟度と将来展望

### 現在の位置づけ（2026年時点）

- OpenAIが「ハーネスエンジニアリング」という用語を広め、業界に急速に浸透
- AI Engineer Europe 2026で専用トラックが設けられるなど、カンファレンスレベルで認知
- 学術研究も始まっている（AGENTS.mdの効率向上を定量的に検証した論文が複数）
- 企業事例が蓄積中（Spotify: 1,500+ PR、Stripe: 週1,300+ PR）

### 将来的な変化の可能性

```
不要になりうるもの:
  - 細かなフォーマット指示（モデルが学習で吸収する可能性）
  - 一部のリンターレベルの品質保証（モデル精度の向上で不要に）
  - セッション間の状態管理の手動設計（ツール側が標準化する可能性）

残り続けるもの:
  - 決定論的な品質保証（テスト・CI）の仕組み
  - 「やってはいけないこと」の機械的な強制
  - ドメイン固有のルールの伝達手段
```

### 本ガイドへの影響

- **シナリオ・テスト中心の成果物は価値が持続する。** 自動検証ループの基盤になる
- **記述的ドキュメントは、検証機構との接続が課題。** 仕様とコードの乖離を検出する仕組みは手動のまま（`01_philosophy.md` の「仕様の腐敗防止」で認めている課題）
- **CLAUDE.md / AGENTS.md から本ガイドの成果物へポインタを張ることで、両者が相互補完する**

---

## 文献リスト

### 原典・定義

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/) | OpenAI (Trivedy / Lopopolo) | 2026.02 | 3名で100万行・1,500PRの事例。ハーネスエンジニアリングの用語を業界に広めた原典 |
| [My AI Adoption Journey](https://mitchellh.com/writing/my-ai-adoption-journey) | Mitchell Hashimoto | 2026.02 | 「エージェントがミスしたら環境を改善する」の定義を初めて明文化。6段階の採用フレームワーク |
| [Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) | Birgitta Bockeler (Thoughtworks) | 2026.02 | 3カテゴリ分類（コンテキスト・アーキテクチャ制約・メンテナンスエージェント） |
| [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Justin Young (Anthropic) | 2025.11 | Initializer Agent + Coding Agentの二段構成。JSON > Markdown。Gitによるセッション引継ぎ |
| [The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/) | Vivek Trivedy (LangChain) | 2026.03 | Agent = Model + Harness の定式化。6つの構成要素に分解 |
| [The importance of Agent Harness in 2026](https://www.philschmid.de/agent-harness-2026) | Philipp Schmid | 2026.01 | ハーネスをエージェントのOSと位置づけ。モジュール式で使い捨て可能な設計を推奨 |

### 学術研究

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [On the Impact of AGENTS.md Files on the Efficiency of AI Coding Agents](https://arxiv.org/abs/2601.20404) | Lulla et al. | 2026.01 | 124 PRで検証。AGENTS.mdにより出力トークン約16.6%削減、処理時間約28.6%短縮 |
| [Evaluating AGENTS.md: Are Repository-Level Context Files Helpful?](https://arxiv.org/abs/2602.11988) | Gloaguen et al. (ETH Zurich) | 2026.02 | **LLM生成の指示ファイルは成功率を2-3%低下させ、コストを20-23%増加させる。** 人間が書いたファイルは+4%の改善 |
| [Building Effective AI Coding Agents for the Terminal](https://arxiv.org/abs/2603.05344) | Nghi D. Q. Bui | 2026.03 | OPENDEVの設計論文。計画と実行の分離、適応的コンテキスト圧縮、イベント駆動型リマインダー |

### 企業事例

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [Background Coding Agents: Predictable Results Through Strong Feedback Loops](https://engineering.atspotify.com/2025/12/feedback-loops-background-coding-agents-part-3) | Max Charas, Marc Bruggmann (Spotify) | 2025.12 | 多段検証パイプライン。LLMジャッジが25%を却下、うち50%が自動修正に成功。1,500+ PRマージ |
| [Minions: one-shot, end-to-end coding agents](https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents) | Stripe Engineering | 2026.02 | 週1,300+ PRを自動マージ。隔離VM・3段階フィードバック・約500ツールの中央MCPサーバー |
| [2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report) | Anthropic | 2026.03 | エンジニアの役割がコード執筆からエージェント調整へ移行。AIは業務の60%で使用されるが完全委譲は0-20% |

### ツール別実践

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [Automate workflows with hooks](https://code.claude.com/docs/en/hooks-guide) | Anthropic | 継続更新 | Claude Code Hooks公式ドキュメント。PreToolUse/PostToolUse等の設計指針 |
| [How to write a great agents.md](https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/) | Matt Nigh (GitHub) | 2025.11 | 2,500+リポジトリの分析。6つの核心領域と3層境界システム |
| [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) | Anthropic | 継続更新 | CLAUDE.md公式推奨。200行以下、各行が「削除したらミスが起きるか」で判断 |
| [claude-code-config](https://github.com/trailofbits/claude-code-config) | Trail of Bits | 継続更新 | セキュリティ重視のClaude Code設定。3段階サンドボックス |
| [Skill Issue: Harness Engineering for Coding Agents](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents) | Kyle (HumanLayer) | 2026.03 | 6つの設定レバー。バックプレッシャー機構が最もレバレッジが高いとの分析 |

### 日本語文献

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [Claude Code / Codex ユーザーのためのHarness Engineeringベストプラクティス](https://nyosegawa.com/posts/harness-engineering-best-practices-2026/) | 逆瀬川ちゃん (@gyakuse) | 2026.03 | 7原則の体系化。言語別ツールスタック。MVHロードマップ。日本語で最も包括的 |
| [AIエージェント時代のハーネスエンジニアリングとは](https://speakerdeck.com/tame/aiezientoshi-dai-nohanesuenziniaringutoha) | 為安圭介 | 2026.03 | 3つの柱: コンテキスト管理・アーキテクチャ制約・多面的品質フィードバック |
| [ハーネスで縛れ、AIに任せろ](https://developers.gmo.jp/technology/81389/) | 平野空暉 (GMO) | 2026.03 | 4段階エスカレーション（ドキュメント → AIセマンティック → CIツール → 構造テスト） |
| [AIコーディングにおけるガードレール](https://zenn.dev/devneko/articles/e88a7786abb86c) | Shinji Yamada | 2025.05 | ハーネスエンジニアリング以前の15のガードレール手法 |
| [Claude CodeのHooksは設定したほうがいい](https://syu-m-5151.hatenablog.com/entry/2025/07/14/105812) | nwiizo | 2025.07 | Hooks導入の動機と実践例。問題検出の左シフト |
| [効果的なCLAUDE.mdの書き方](https://zenn.dev/farstep/articles/how-to-write-a-great-claude-md) | farstep | 2026.02 | 500行以内推奨。3層構造: コアCLAUDE.md + .claude/rules/ + Skills |
| [スターの多いOSSは、どのようなAGENTS.mdを記述しているのか](https://developers.freee.co.jp/entry/oss-agents-md) | Shiba (freee) | 2025.12 | 人気OSSのAGENTS.md実例分析。理想は60-300行/25KB以下 |
| [AIエージェントの内部構造 — エージェントハーネスの概要と設計・実装](https://codezine.jp/article/detail/23340) | 西見公宏 (CodeZine) | 2026.03 | ハーネスをOSに例えた解説。コンテキスト管理・ツール実行管理・タスク計画の3役割 |

### 業界分析・レポート

| 文献 | 著者 | 年月 | 概要 |
|------|------|------|------|
| [DORA State of AI-assisted Software Development 2025](https://dora.dev/dora-report-2025/) | Google DORA Team | 2025 | AI導入率90%だが組織レベルのデリバリー指標は横ばい。AIは組織の既存の強みと弱みを増幅する |
| [2025 Was Agents. 2026 Is Agent Harnesses.](https://aakashgupta.medium.com/2025-was-agents-2026-is-agent-harnesses-heres-why-that-changes-everything-073e9877655e) | Aakash Gupta | 2026 | エンジニアの仕事が「コードを書く」から「エージェントが正しく動く環境を作る」へ移行 |
| [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) | Michael Bolin (OpenAI) | 2025 | モデルとハーネスは共設計される（独立ではない）。ハーネスの存在下でモデルが訓練されている |
