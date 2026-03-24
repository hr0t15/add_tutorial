# Appendix G: AIとの協調開発がなぜ機能するのか — 外部調査・報告からの裏付け

このガイドでは「人間が上流で判断を構造化し、AIに委譲する」というプロセスを採用している。
このAppendixでは、そのアプローチがなぜ有効なのかを外部の調査・報告から裏付ける。

なお、ここで引用する情報源の性質はさまざまである。

| 種類 | 例 | 特徴 |
|------|---|------|
| **大規模サーベイ** | DORA 2025 | 数千人規模の開発者を対象にした定量調査。統計的に有意な知見を含む |
| **業界調査・分析** | GetDX, StackAI | 企業のAI導入状況をまとめた業界レポート。定性的な傾向を示す |
| **企業の社内テスト** | IBM | 特定企業の内部テスト結果。条件が限定的だが実務に近い数字を提供する |
| **解説記事** | IT Revolution, Faros AI | 上記の調査を解釈・要約した二次情報 |

以下では、出典の種類を明示しながら記述する。

---

## AIは組織の鏡である（Mirror Effect）

**出典: DORA 2025（大規模サーベイ）**

Google の DORA（DevOps Research and Assessment）が2025年に発表した報告書の中核メッセージは明快である。

> AIは組織を直すのではなく、すでにあるものを増幅する。
> 強いチームはAIでさらに強くなり、弱いチームはAIで弱さが露呈する。

つまりAIを導入するだけでは成果は出ない。**AIが効果を発揮するには、AIに渡せる形で知識が構造化されている必要がある。**

このガイドの要求定義の構造（要件概要 → 振る舞い定義 → ドメイン定義 → 判断記録）は、開発プロセスの中で自然にこの「構造化された知識」を作り出す仕組みである。

---

## AIの効果を組織に拡大する条件

**出典: DORA 2025（大規模サーベイ）**

DORA 2025は「AIの効果を個人レベルから組織レベルに拡大する7つのケイパビリティ」を特定している。そのうち、このガイドと特に関連が深いものを整理する。

| ケイパビリティ | 内容 | このガイドとの対応 |
|--------------|------|--------------------|
| **AI-accessible internal data** | AIが社内のドキュメント・コードベース・判断記録にアクセスできること | 要求定義・判断記録に判断と理由を残す設計 |
| **Healthy data ecosystems** | 高品質で統一されたデータが整備されていること | ユビキタス言語による用語統一、構造化されたドキュメント群 |
| **Working in small batches** | 小さい単位で作業すること | feature単位の委譲、コンテキスト設計 |

「AI-accessible internal data」は、個人の生産性とコード品質の両方を**統計的に有意に向上させる**と報告されている。ここで言う「internal data」とは単なるドキュメントではなく、**AIに渡せる形で構造化された社内知識**を指す。

---

## 生産性パラドックス: 個人は速くなっても組織は速くならない

**出典: DORA 2025（大規模サーベイ）+ Faros AI（解説記事）**

DORA 2025は興味深いデータを示している。

```
個人レベル: タスク完了 +21%、PRマージ +98%
組織レベル: デリバリー指標は横ばい
```

個人は確実に速くなっている。にもかかわらず、組織全体の成果には反映されていない。

これは「AIに何を渡すか」の設計ができていない組織では、個人の速さが組織の品質に転換されないことを示唆している。このガイドが「フェーズ1で何を・なぜ作るかを固める」ことに最も時間をかけるべきとしているのは、この問題への対処でもある。

---

## コンテキストエンジニアリングという考え方

**出典: DORA 2025 + Google Cloud Blog（解説記事）**

DORA報告書では、AIに渡す文脈をシステムとして設計する手法を**コンテキストエンジニアリング**と呼んでいる。単にプロンプトを工夫するのではなく、AIが参照すべき情報を構造的に整備する取り組みである。

このガイドの `01_philosophy.md` で述べている「コンテキストを設計するという考え方」は、同じ発想に基づいている。

---

## ファイル単位のコンテキストでは不十分

**出典: GetDX（業界調査）**

GetDXの調査では、ファイル単位のコンテキストでは実際のコードベースで不十分であり、リポジトリ横断の認識・サービス境界・歴史的な依存関係の理解が必要とされている。

パイロットでは上手くいくが、本番の共有リポジトリ・チームレビュー・CI/CDで差が出る。このガイドが単一ファイルではなく要求定義全体を構造化する理由はここにある。

---

## 95%のパイロットが実験段階を超えられない

**出典: StackAI（業界調査）**

StackAIの調査によると、生成AIパイロットの95%が実験段階を超えられないとされている。失敗の原因は技術ではなく、組織・プロセスの問題として扱えていないことにある。

AIの導入は「どのツールを使うか」ではなく「どうプロセスに組み込むか」で成否が決まる。

---

## ビジネスドメインの知識はAIが生成できない

**出典: IBM（企業の社内テスト）**

IBMの社内テストでは、AIによるコードドキュメンテーション作成時間が平均59%短縮されたと報告されている。技術的なドキュメントはAIに任せられる時代になりつつある。

ただし、**「なぜ作るか」「何が重要か」というビジネスドメインの知識はAIが生成できない**。これが、このガイドのフェーズ1で人間主導を強調し、要件概要や判断記録に「なぜ」を残すことを求めている根拠である。

---

## 文献リスト

### 大規模サーベイ

- [DORA State of AI-assisted Software Development 2025](https://dora.dev/research/2025/dora-report/) — 数千人規模の開発者調査。AIは組織の既存の強さを増幅する（Mirror Effect）。組織レベルの成果に転換するには7つのケイパビリティが必要
- [DORA Capabilities: AI-accessible internal data](https://dora.dev/capabilities/ai-accessible-internal-data/) — AIが社内の構造化された知識にアクセスできることが、生産性とコード品質を統計的に有意に向上させる

### 業界調査

- [AI code generation: Best practices for enterprise adoption](https://getdx.com/blog/ai-code-enterprise-adoption/) — GetDX。ファイル単位のコンテキストでは不十分。リポジトリ横断の認識が必要
- [Enterprise AI Adoption 2026: Trends, Benchmarks, and Best Practices](https://www.stackai.com/insights/enterprise-ai-adoption-2026-trends-benchmarks-and-best-practices-for-scalable-success) — StackAI。生成AIパイロットの95%が実験段階を超えられない

### 企業の社内テスト・報告

- [AI Code Documentation: Benefits and Top Tips](https://www.ibm.com/think/insights/ai-code-documentation-benefits-top-tips) — IBM。社内テストでAIによるドキュメンテーション作成時間が平均59%短縮
- [AI coding is now everywhere. But not everyone is convinced.](https://www.technologyreview.com/2025/12/15/1128352/rise-of-ai-coding-developers-2026/) — MIT Technology Review。AI導入の普及と懐疑的な声の背景

### 解説記事

- [AI's Mirror Effect: How the 2025 DORA Report Reveals Your Organization's True Capabilities](https://itrevolution.com/articles/ais-mirror-effect-how-the-2025-dora-report-reveals-your-organizations-true-capabilities/) — IT Revolution。DORA 2025の「鏡効果」の詳細な解説
- [DORA Report 2025 Key Takeaways](https://www.faros.ai/blog/key-takeaways-from-the-dora-report-2025) — Faros AI。生産性パラドックス（個人+21%/+98%、組織横ばい）の分析
- [From adoption to impact: Putting the DORA AI Capabilities Model to work](https://cloud.google.com/blog/products/ai-machine-learning/from-adoption-to-impact-putting-the-dora-ai-capabilities-model-to-work) — Google Cloud Blog。コンテキストエンジニアリングの重要性とDORA AI Capabilities Modelの実践方法
