---
description: "[Flow初期化] 仕様書からTDDフローを初期化（タスク自動生成）"
allowed-tools: Read, Write, Edit, Bash, Glob
argument-hint: "<feature-name> [--reset]"
---

# Flow初期化

仕様書（design.md）からタスクを自動生成し、TDDフローを初期化します。

**必須**: feature-name引数（仕様書名）

## 実行フロー

**コマンド**: `/flow-init <feature-name>`

### ステップ1: 仕様書の確認

1. `.claude/specs/$1/spec.json`の存在確認
2. `.claude/specs/$1/design.md`の存在確認

**エラーハンドリング**:

- `spec.json`が存在しない → `/spec-init`の実行を案内
- `design.md`が存在しない → `/spec-design`の実行を案内

### ステップ2: タスクの自動生成

設計書（`design.md`）から実装タスクを自動生成します。

**タスク生成ルール**:

- 自然言語で「何をするか」を記述（ファイル名やコード構造ではなく、機能と成果に焦点）
- 各タスクは1〜3時間で完了可能
- メジャータスク: 1, 2, 3, 4...（逐次的に番号付け）
- サブタスク: 1.1, 1.2, 2.1, 2.2...（メジャータスクごとにリセット）
- 最大2レベルの階層（1.1.1のような深い階層は不可）

**タスクフォーマット**:

```text
# 実装タスクリスト: [feature-name]

- [ ] 1. [メジャータスク1の説明]
- [ ] 1.1 [サブタスク1.1の説明]
  - 詳細1
  - 詳細2
  - _Requirements: 1.1, 1.2_

- [ ] 1.2 [サブタスク1.2の説明]
  - 詳細1
  - _Requirements: 1.3_

- [ ] 2. [メジャータスク2の説明]
- [ ] 2.1 [サブタスク2.1の説明]
  - 詳細1
  - 詳細2
```

**タスク統合と進行**:

- 各タスクは前のタスクの出力に基づいて構築
- 統合タスクで全体を接続
- 孤立した機能を作らない
- 段階的な複雑さ（タスク間の大きなジャンプを避ける）

### ステップ3: state.jsonの初期化

既存の`.claude/.flow/`が存在し、`--reset`オプションがない場合は警告を表示して終了します。

`.claude/.flow/`ディレクトリを作成し、`state.json`を初期化：

```jsonc
{
  "phase": "red",
  "current_task": "[最初のタスク名]",
  "completed_tasks": [],
  "pending_tasks": ["[残りのタスクリスト]"],
  "history": [],
  "spec": {
    "feature_name": "[feature-name]",
    "spec_path": ".claude/specs/[feature-name]/",
    "has_design": true,
    "has_requirements": [requirementsの有無]
  }
}
```

**フィールド説明**:

- `spec`: 仕様書情報
  - `feature_name`: 機能名
  - `spec_path`: 仕様書のパス
  - `has_design`: 設計書の有無（trueで固定）
  - `has_requirements`: `requirements.md`の存在に応じて設定

### ステップ4: tasks.mdの保存

`.claude/.flow/tasks.md`にタスクリストを保存します。

### 出力

```text
✅ TDDフローを初期化

📦 仕様書: [feature-name]
   - spec.json ✅
   - design.md ✅
   - requirements.md [✅/❌]

📋 生成されたタスク: [N]個
🎯 最初のタスク: 1. [タスク名]

次のステップ: /flow-run または /flow-next
```

---

## オプション

| オプション | 内容 |
|-------------|------|
| `--reset` | `.claude/.flow`をバックアップして再初期化 |

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| feature-name引数なし | エラー: 必須引数（feature-name）が指定されていません |
| `spec.json`が存在しない | `/spec-init`の実行を案内 |
| `design.md`が存在しない | `/spec-design`の実行を案内 |
| `.claude/.flow/`既存・`--reset`なし | 初期化中止、警告出力 |
| ファイル書込不可 | エラー終了 |

---

## 使用例

```bash
# 仕様書からTDDフローを初期化
/flow-init user-auth

# 再構築（既存データをバックアップ）
/flow-init user-auth --reset
```
