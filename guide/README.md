# guide/ — 思想・プロセス・リファレンス

AIとの協調開発における「なぜこのプロセスか」「どう進めるか」をまとめたドキュメント群。

チュートリアルを進めながら辞書的に参照する使い方を想定している。

---

## 読む順番

初めて読む場合は以下の順序を推奨する。

```
01_philosophy.md          ← まずここから
  ↓
02_development_process.md ← 全体フローを理解
  ↓
03_directory_structure.md ← ディレクトリの設計思想
  ↓
04_collaboration_matrix.md ← フェーズ別の役割分担
  ↓
05_bdd_and_ddd.md         ← BDD・DDDの活用方法
  ↓
06_ui_design.md           ← UI設計の共通原則（GUI / CLI）
06_01_gui.md              ← GUI固有の設計・テスト戦略
06_02_cli.md              ← CLI固有の設計・テスト戦略
  ↓
07_modification_process.md ← 改修時の進め方
```

Appendix は必要に応じて参照する。

---

## メインドキュメント

| ファイル | 内容 | 読むタイミング |
|---------|------|-------------|
| `01_philosophy.md` | 基本思想・人間とAIの役割 | 最初に必ず読む |
| `02_development_process.md` | 開発プロセス全体フロー | 最初に必ず読む |
| `03_directory_structure.md` | ディレクトリ構成と設計思想 | プロジェクト開始時 |
| `04_collaboration_matrix.md` | 協調マトリックス・規模別プラクティス | フェーズの進め方に迷ったとき |
| `05_bdd_and_ddd.md` | BDD・DDDの活用 | シナリオ・ドメインモデルを書くとき |
| `06_ui_design.md` | UI設計の共通原則 | UIの設計・テストを進めるとき |
| `06_01_gui.md` | GUI設計とテスト戦略 | 画面のあるアプリケーションを作るとき |
| `06_02_cli.md` | CLI設計とテスト戦略 | コマンドラインツールを作るとき |
| `07_modification_process.md` | 改修プロセス | 既存システムを変更・更新するとき |

## Appendix

メインドキュメントの実践時に、具体的な書き方・進め方を確認するためのリファレンス。

| ファイル | 内容 | 読むタイミング | 関連するメインドキュメント |
|---------|------|-------------|----------------------|
| `appendix/A_bdd.md` | Gherkin形式・featureファイルの書き方 | featureファイルを初めて書くとき | `02`, `05`, `06_ui_design` |
| `appendix/B_ddd.md` | ユビキタス言語・ドメインモデルの作り方 | domain/ のファイルを作るとき | `04`, `05` |
| `appendix/C_event_storming.md` | イベントストーミングの進め方 | イベントストーミングを実施するとき | `02`, `04`, `05` |
| `appendix/D_adr.md` | ADRのテンプレート・書き方 | ADRを書くとき | `03`, `04` |
| `appendix/E_rdra.md` | RDRA（リレーションシップ駆動要件分析）との関係 | 要件間のトレーサビリティを強化したいとき | `02`, `03` |
| `appendix/F_context_engineering.md` | コンテキストエンジニアリング実践ガイド | AIに渡す情報の設計・設定ファイルの活用 | `01`, `02`, `04` |
| `appendix/G_ai_adoption_insights.md` | AIとの協調開発の研究的裏付け | このアプローチがなぜ有効か知りたいとき | `01` |
| `appendix/H_spec_driven_development.md` | スペック駆動開発の展望 | 仕様起点の開発アプローチの最新動向を知りたいとき | `01`, `02` |
| `appendix/I_harness_engineering.md` | ハーネスエンジニアリング | コーディングAIの実行環境を設計するとき | `01`, `02` |
