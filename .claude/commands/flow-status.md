---
allowed-tools: Read(*.md)
description: "現在の状態（phase/current_task/next_tasks/最近のテストとログ）を要約します。--json/--detail で出力形式を切替。"
---

あなたは **ステータス表示コマンド** です。引数が無ければ人間可読のMarkdownを表示します。

## 任意引数

- `--json` : 構造化JSONを出力（CIや外部ツール連携用）
- `--detail` : 全タスク/全ログを展開
- `--phase-only` : phase と current_task のみ簡潔表示

## 取得対象

- `.claude/flow/state.json`  
- 直近 `pytest` の実行結果（キャッシュ/ログがあれば）
- 直近 `codex review/tests` の要約（あれば）

## 出力形式（markdown）

```
【現在の状態】
- Phase: red|green|refactor
- Current Task: ...
- Next Tasks(Top5): ...

【テスト結果】
- pytest: 成功/失敗（要約）
- カバレッジ（あれば）: ...

【最近ログ】
- [時刻] init: ...
- [時刻] next: phase=green → success
- ...

【推奨次アクション】
- flow-next
- flow-run --max-cycles 3
```

## 出力形式（--json）

```json
{
  "phase": "green",
  "current_task": "...",
  "next_tasks": ["...", "..."],
  "pytest": {"success": true, "failed_tests": []},
  "coverage": {"overall": 0.86},
  "recent_logs": [{"ts":"...","kind":"...","msg":"..."}]
}
```

## 注意

- state が無い場合は flow-init を促す  
- 情報が欠けている要素（カバレッジ等）は省略/空で返す  
