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
