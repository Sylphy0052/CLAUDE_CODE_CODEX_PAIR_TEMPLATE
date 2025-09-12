---
allowed-tools: Bash(git:*), Read(*.md), Fetch(*)
description: "初期化。docs/spec.md を読み、調査→計画→タスク分割→state / tasks / config ファイル作成までを自動実行します。"
updated: "2025-09-12"
---

# flow-init

あなたは **初期化コマンド (flow-init)** です。プロジェクトを TDD サイクルで回すための**下準備**を行います。
ここでは Spec Writer と Planner のサブエージェントを使い、調査と計画、およびタスク分割・状態ファイル・設定ファイルの生成のみを行います。

---

## デフォルト挙動（引数なし）

1. **情報源の探索（Spec Writer サブエージェント）**
   - 優先: `docs/spec.md`
   - 他: `README.md` / `docs/*.md` / `SPEC.md` / `PRD.md` / `TODO.md` / `CHANGELOG.md` / コードコメント
   - 見つからなければ最小 SPEC の雛形を提案
   - 出力は **Markdown 要約** と **JSON フォーマット（research_digest）** の両方を生成

2. **調査結果の取り込み**
   - Spec Writer の出力から `research_digest` を抽出
   - `goal`, `non_goals`, `constraints`, `open_questions`, `source_files` を state.json に格納

3. **計画（Planner サブエージェント）**
   - `research_digest` をもとにタスクを分割
   - 各タスクは **2時間以内** に完了可能な粒度
   - 必ずテストで検証可能な形にする
   - 各タスクには以下を含めること:
     - `id`: `T{連番}-{slug}` 形式（slug は小文字・英数字・ハイフン）
     - `priority`: 1–10（必須）
     - `depends_on`: 他タスクID（循環依存は禁止）
     - `milestone`: 紐付けマイルストーン
     - `acceptance_criteria`: Given/When/Then
     - `context`: spec_refs / related_files
   - Planner の出力は **Markdown 要約** と **JSON タスクリスト** の両方を生成

4. **ファイル生成**
   - `.claude/flow/state.json`（現在の状態）
   - `.claude/flow/tasks.json`（タスク管理）
   - `.claude/flow/config.json`（存在しなければ生成）
   - `docs/tasks.md`（上部: 一覧表、下部: 各タスク詳細セクション）
   - 既存があれば `.bak-<timestamp>` に退避

---

## 任意引数

- `--goal "<text>"` : 目的テキストを直接指定（spec より優先）
- `--spec "<file>"` : 仕様ファイルを明示指定（既定: `docs/spec.md`）
- `--reset` : 既存 state を無視して再生成（.bak に退避）

---

## 出力形式（人間向け要約）

- 【Research 要点（Markdown）】 …
- 【Research Digest（JSON）】 …
- 【Plan 要点（Markdown）】 …
- 【タスクリスト（JSON）】 …
- 【state 要約】 phase=planned / current_task / next_tasks / warnings

---

## 生成ファイル例

### `.claude/flow/state.json`

```json
{
  "schema_version": "1.0",
  "phase": "planned",
  "goal": "<短い目的>",
  "research_digest": {
    "goal": "...",
    "non_goals": ["..."],
    "constraints": ["..."],
    "open_questions": ["..."],
    "source_files": ["docs/spec.md","README.md"]
  },
  "plan": {
    "milestones": ["..."],
    "acceptance_criteria": ["Given/When/Then ..."],
    "risks": ["..."]
  },
  "current_task": "T001-login-feature",
  "next_tasks": ["T002-session-store", "T003-rate-limit"]
}
```

### `.claude/flow/tasks.json`

```json
{
  "id": "T001-login-feature",
  "title": "Login feature",
  "status": "planned",
  "priority": 8,
  "depends_on": [],
  "milestone": "M1 Authentication",
  "acceptance_criteria": [
    {"given": "ユーザーが有効な資格情報を入力", "when": "ログインを実行", "then": "セッションが開始される"}
  ],
  "context": {
    "spec_refs": ["docs/spec.md#auth"],
    "related_files": ["src/auth/*.ts"]
  },
  "history": [],
  "created_at": "<ISO8601>",
  "updated_at": "<ISO8601>"
}
```

### `.claude/flow/config.json`

```json
{
  "schema_version": "1.0",
  "default_milestones": ["M1 Authentication"],
  "id_format": "T{num:03}-{slug}",
  "task_slug_rules": "lowercase, hyphen-separated, alphanumeric only",
  "backup_dir": ".claude/backups"
}
```

---

## 注意事項

- flow-init は **Spec Writer と Planner の2つのサブエージェント**を必ず使用する
- 初期化直後のフェーズは常に **planned**
- 実装・レビュー・テストはここでは行わない（flow-next / flow-run に委譲）
