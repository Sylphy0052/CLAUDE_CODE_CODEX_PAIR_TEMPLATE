---
description: "[仕様初期化] 新規機能の仕様を初期化"
allowed-tools: Read, Write, Glob, Bash
argument-hint: <feature-description>
---

# 仕様初期化

新規機能の仕様を`.claude/specs/[feature-name]/`に作成します。

**機能説明**: $ARGUMENTS

## Task

### 1. 機能名の生成

`$ARGUMENTS`からケバブケース形式の機能名を生成します。

**生成ルール**:
- 英数字とハイフンのみ
- 2〜4単語程度
- 例: 「ユーザー認証機能」→「user-auth」

**重複チェック**: `.claude/specs/`を確認し、同名が存在する場合は番号を付与（例: `user-auth-2`）

### 2. ディレクトリ作成

`.claude/specs/[feature-name]/`を作成します。

```bash
mkdir -p .claude/specs/[feature-name]
```

### 3. spec.jsonの作成

以下の内容で`spec.json`を作成：

```json
{
  "feature_name": "[feature-name]",
  "description": "$ARGUMENTS",
  "created_at": "[ISO8601 timestamp]",
  "updated_at": "[ISO8601 timestamp]",
  "language": "ja",
  "phase": "initialized",
  "has_requirements": false,
  "has_design": false
}
```

**フィールド説明**:
- `feature_name`: 生成した機能名
- `description`: ユーザーが入力した機能説明（$ARGUMENTS）
- `created_at`: 作成日時（ISO8601形式）
- `updated_at`: 更新日時（ISO8601形式）
- `language`: 言語設定（"ja"固定）
- `phase`: 現在のフェーズ（"initialized"から開始）
- `has_requirements`: 要件定義書の有無（falseから開始）
- `has_design`: 設計書の有無（falseから開始）

### 4. タイムスタンプの生成

ISO8601形式のタイムスタンプを生成：

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

## 次のステップ

仕様駆動開発のワークフローに従い、以下の順で進めます：

1. **`/spec-requirements [feature-name]`** - 要件定義書を生成（EARS形式）
2. **`/spec-design [feature-name]`** - 設計書を生成（詳細設計）
3. **`/flow-init [feature-name]`** - 実装を開始（タスク自動生成 + TDD初期化）

## 出力

```
✅ 仕様を初期化

📦 機能名: [feature-name]
📝 説明: [機能説明]
📄 作成: .claude/specs/[feature-name]/spec.json

次のステップ: /spec-requirements [feature-name]
```

## エラーハンドリング

- `$ARGUMENTS`が空の場合: エラーメッセージを表示し、機能説明の入力を促す
- ディレクトリ作成失敗: 権限エラーまたはパスエラーを表示
- JSON書き込み失敗: エラー内容を表示

## 注意事項

- この段階では`spec.json`のみを作成します
- `requirements.md`と`design.md`は後続のコマンドで生成されます
- 段階的な開発原則に従い、一度に1つのステップのみを実行します
