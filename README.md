# CLAUDE CODE CODEX PAIR TEMPLATE

Claude CodeとCodex CLIを併用したTDD開発テンプレートプロジェクト

## 概要

このプロジェクトは、Claude CodeとCodex CLIを並行利用し、テスト駆動開発（TDD）を実践するためのテンプレートです。Red→Green→Refactorサイクルを徹底し、相互レビューによる品質向上を目指します。

## 特徴

- **TDD徹底**: すべての機能はテストファーストで開発
- **相互レビュー**: Claude CodeとCodex CLIによる双方向レビュー
- **パッチベース開発**: 全変更を差分提案→承認→適用で管理
- **自動化**: テスト、Lint、型チェックの自動実行

## プロジェクト構造

```
.
├── .claude/           # Claude Code設定
├── .codex/            # Codex CLI設定
├── docs/              # ドキュメント
├── AGENTS.md          # エージェント定義
├── CLAUDE.md          # Claude Code運用規約
└── HOW_TO_USE.md      # 使用方法ガイド
```

## セットアップ

```bash
# Poetry環境のセットアップ
poetry install --no-root

# 開発環境の初期化
poetry run python main.py
```

## 開発ワークフロー

1. **flow-init**: 目標読込・タスク分解・Redテスト計画
2. **tests-red**: 失敗テストを追加
3. **tests-green**: 最小実装で成功させる
4. **tests-refactor**: 外部挙動を変えず整理
5. **reviewer/codex-review**: 相互レビュー
6. **flow-next**: 次タスクへ継続

## コマンド

### テスト実行

```bash
poetry run pytest
```

### コード品質チェック

```bash
# Lintチェック
poetry run flake8 --max-line-length=120 --ignore=E203,E231,E402,W503,W605 .

# 型チェック
poetry run mypy --strict .
```

## 要件

- Python 3.11.6+
- Poetry
- Claude Code CLI
- Codex CLI

## ライセンス

このプロジェクトはテンプレートとして提供されています。

## 仕様書ベースの実装フロー

spec.md（`docs/spec.md`）に基づいて実装を進める際の標準フローです。

### 1. 実装開始前の準備

```bash
# spec.mdの内容を確認
cat docs/spec.md

# 実装タスクをspec.mdから抽出
/flow-init --spec docs/spec.md
```

### 2. TDD実装サイクル

#### Phase 1: Red（失敗テスト作成）

```bash
# spec.mdの要件からテストケースを生成
/tests-red --spec docs/spec.md --section "4. データモデル"

# 生成されたテストを確認
poetry run pytest -xvs tests/ -k "test_models"
```

#### Phase 2: Green（最小実装）

```bash
# spec.mdに準拠した最小実装を生成
/tests-green --spec docs/spec.md

# テストが通ることを確認
poetry run pytest
```

#### Phase 3: Refactor（リファクタリング）

```bash
# コード品質を改善（外部挙動は変更しない）
/tests-refactor

# Lint/型チェック
poetry run flake8 --max-line-length=120
poetry run mypy --strict
```

### 3. 相互レビュー

```bash
# Claude Codeでレビュー
/reviewer --spec docs/spec.md

# Codex CLIでレビュー（WSL環境）
/codex-review --spec docs/spec.md

# レビュー結果の比較・マージ
/codex-bridge --compare
```

### 4. 実装の進捗管理

```bash
# 現在の実装状況を確認
/flow-status --spec docs/spec.md

# 次のタスクへ進む
/flow-next
```

### 5. spec.mdとの整合性チェック

```bash
# 実装がspec.mdの要件を満たしているか検証
poetry run pytest tests/spec_compliance_test.py

# ドキュメントとコードの乖離をチェック
/doc-smith --validate docs/spec.md
```

### 実装例: MVPスコープ（spec.md Section 14）

```bash
# 1. プロジェクト/章/QAモデルの実装
/flow-init --spec docs/spec.md --section "4. データモデル"
/tests-red --models Project,Outline,Chapter,QAItem
/tests-green
/tests-refactor

# 2. FastAPI エンドポイントの実装
/flow-init --spec docs/spec.md --section "5. API設計"
/tests-red --api "POST /projects"
/tests-green
/codex-review

# 3. Codex/Claude Runner実装
/flow-init --spec docs/spec.md --section "6. ランナー/アダプタ仕様"
/scaffold --template runner --spec docs/spec.md
/tests-red --adapter codex_wsl,claude_code
/tests-green

# 4. Docker環境構築
/scaffold --docker --spec docs/spec.md --section "13. Docker"
```

### spec.md更新時の対応

```bash
# spec.mdが更新された場合
git diff docs/spec.md

# 変更箇所に対応するテストを更新
/tests-red --spec docs/spec.md --changed

# 実装を更新
/tests-green --spec docs/spec.md --changed

# ドキュメント同期
/doc-smith --sync docs/spec.md
```

### トラブルシューティング

```bash
# spec.mdとの不整合を検出
/spec-writer --validate

# 実装とspec.mdの差分を可視化
/reviewer --diff-spec docs/spec.md

# spec.mdに基づく修正提案
/planner --fix-compliance docs/spec.md
```
