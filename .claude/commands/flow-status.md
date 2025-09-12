---
allowed-tools: Read(*.md)
description: "現在の状態（phase/current_task/next_tasks/last_diff 等）を要約表示。出力オプション/依存DAG検証/アンカー連携に対応。state がなければ flow-init を促します。"
updated: "2025-09-12"
---

# flow-status

`.claude/flow/state.json` を読み取り、プロジェクトの現在地を**簡潔に**表示します。
未初期化の場合は `flow-init` の実行を案内します。詳細運用は `flow-run` を参照してください。

---

## 使い方

- `flow-status`
- `flow-status --format json`
- `flow-status --compact`

### 出力オプション

- `--format <md|json>` : 出力形式を選択（既定: `md`）
- `--compact` : 主要項目のみ表示（補足・警告の詳細を省略）

---

## 出力（Markdown, 既定）

【現在の状態】

- phase: `<planned|red|green|refactor|reviewing|testing|done>`
- current_task: `<T###-slug or none>`
- next_tasks (max 5):
  1. `[T001](docs/tasks.md#t001-login-feature)` 例
  2. ...
- last_diff（要約/あればのみ）:

  ```diff
  <直近の変更点の抜粋>
  ```

- warnings:
  - ...

【補足】（必要に応じて）

- research_digest（要点）: ...
- plan 概要（milestones / acceptance_criteria / risks）: ...
- config（id_format / default_milestones など）: ...
- updated_at: `<ISO8601>`

> 注: `phase=planned` のときは **`last_test_result` を表示しません（N/A）**。

---

## 出力（JSON 例）

```json
{
  "phase": "planned",
  "current_task": "T001-login-feature",
  "next_tasks": [
    {"id": "T002-session-store", "anchor": "docs/tasks.md#t002-session-store"},
    {"id": "T003-rate-limit", "anchor": "docs/tasks.md#t003-rate-limit"}
  ],
  "last_diff": null,
  "warnings": [],
  "supplement": {
    "research_digest": "...",
    "plan_summary": {"milestones": 3, "acceptance_criteria": 5, "risks": 2},
    "config": {"id_format": "T{num:03}-{slug}"}
  },
  "updated_at": "2025-09-12T10:00:00+09:00"
}
```

`--compact` 時は `next_tasks` を最大3件、`supplement` を省略します。

---

## 動作ルール

- **state.json が無い** → 「未初期化。`flow-init [--spec docs/spec.md]` を実行してください」と表示
- **DAG検証**（`tasks.json` がある場合のみ）
  - `depends_on` の **循環**・**孤立タスク** を検出し、`warnings` に要約（例: `cycle: T004→T005→T004` / `orphan: T010`）
- **アンカー連携**
  - `next_tasks` には `docs/tasks.md` の見出しアンカーを付与（例: `[T001](docs/tasks.md#t001-login-feature)`）
- **差分表示**
  - 直近の差分が **ある場合のみ** ```diff ブロックを表示（無ければ省略）
- **テスト表示**
  - `phase=planned` は `last_test_result` を **非表示**
- 表示は**要点のみ**。詳細な工程や操作は **`flow-run` を参照**

---

## 参照ファイル

- `.claude/flow/state.json`（必須）
- `.claude/flow/tasks.json`（DAG検証/アンカー生成に使用）
- `docs/tasks.md`（アンカーリンクの出力に使用）

---
