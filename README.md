# CLAUDE_CODE_CODEX_PAIR_TEMPLATE

このリポジトリは **Claude Code** と **Codex CLI** を組み合わせ、
**調査 → 計画 → 実装 → テスト → レビュー → 承認** の開発サイクルを自動化するテンプレートです。

---

## 特徴

- **Claude × Codex の二重化**
  Codex が実装・テストを行い、Claude Code がレビュー・統合判断を実施
- **TDD 準拠の実装サイクル**
  Red → Green → Refactor の流れをコマンドで自動化
- **サブエージェントによる分業**
  - Spec Writer（仕様整理）
  - Planner（タスク分割）
  - Implementer（実装）
  - Reviewer（レビュー）
  - Doc Writer / Docs Guardian（文書整合）

---

## コマンド概要

- **flow-init**
  初期化。`docs/spec.md` を読み込み、調査 → 計画 → state/tasks ファイル生成

- **flow-status**
  現在の状態（phase / current_task / next_tasks / last_diff など）を要約表示

- **flow-next**
  **サイクル単位**で TDD（Red → Green → Refactor）を進める
  `--cycles N` で複数回実行可能

- **flow-run**
  **マイルストーン or フェーズ単位**でまとめて進め、完了後に総合テスト・相互レビュー・承認を行う

- **flow-reset**
  state とワークツリーを安全に巻き戻す（バックアップから復元可能）

---

## flow-next と flow-run の違い

- **flow-next** : **小刻みに進める**（タスク/サイクル単位）
- **flow-run** : **まとまった単位で進める**（マイルストーン/フェーズ単位）

---

## ワークフロー概要

1. `flow-init` で調査・計画を行い、state.json / tasks.json を生成
2. 開発中は `flow-next` で小刻みに進める
3. 節目では `flow-run` を使い、まとめて検証・承認
4. 状態確認は常に `flow-status`
5. 不整合や異常が起きたら `flow-reset` で巻き戻す

---

## 関連ドキュメント

- [HOW_TO_USE.md](HOW_TO_USE.md) : 日常の具体的な操作手順
- `.codex/config.toml` : Codex の設定ファイル
- `.claude/agents/` : 各エージェントの役割定義
- `.claude/commands/` : コマンド仕様
