---
allowed-tools: Bash(git:*), Bash(python:*), Read(*.md), Fetch(*)
description: "初期化。CLAUDE.mdや既存アーティファクトを読み取り、目的・制約の要約、タスク分解、最初のRedテスト準備、状態ファイル作成まで自動実行します。"
---

あなたは **初期化コマンド** です。引数が無ければ自動で情報源を探索し、TDDフロー開始に必要な最小要素を揃えます。必要なら `--goal` などで上書き指定できます。

## デフォルト挙動（引数なし）

1. **情報源の発見**（優先度順）
   - `.claude/CLAUDE.md` の Purpose/Principles/Workflow 節
   - `README.md` / `docs/` / `SPEC.md` / `PRD.md`
   - 直近のIssue/ToDo（`TODO.md`/`CHANGELOG.md` など）
2. **目的と制約の要約**
   - 目的（Goal）、非目的（Non-Goals）、制約（技術/運用/性能/セキュリティ）を抽出
3. **タスク分解（≤2h単位）**
   - `@.claude/agents/planner.md` を適用し、受入基準（Given/When/Then）付きで分解
4. **最初のRedテスト準備**
   - `tests/` が無ければ作成、`test_<domain>_smoke.py` の叩き台を提案
   - **失敗が保証される**最小テスト（存在しない関数/例外期待 等）
5. **状態ファイルの生成**
   - `.claude/flow/state.json` を作成（スキーマは後述）
   - 初期値: `phase="red"`, `current_task=<First Task>`, `next_tasks=[...]`
6. **安全系**
   - 既に状態がある場合はバックアップ `.claude/flow/state.json.bak-YYYYMMDD-HHMMSS` を作成

## 任意引数（上書き指定）

- `--goal "<text>"` : 目的テキストを直接与える（CLAUDE.mdより優先）
- `--with-scaffold` : `@scaffold` を呼び、最小ディレクトリ/テスト雛形も生成
- `--reset` : 既存 state を破棄して再初期化（バックアップは残す）

## 状態スキーマ（.claude/flow/state.json）

```json
{
  "version": 1,
  "updated_at": "2025-09-10T00:00:00Z",
  "phase": "red", // "red" | "green" | "refactor"
  "current_task": "短い見出し",
  "next_tasks": ["...", "...", "..."],
  "acceptance_criteria": ["Given ... When ... Then ..."],
  "recent_logs": [
    {"ts":"...", "kind":"init", "msg":"initialized by flow-init"}
  ],
  "stats": {"cycles": 0, "failed_cycles": 0, "total_diff_lines": 0}
}
```

## 出力形式

```
【目的・制約要約】
- Goal: ...
- Non-Goals: ...
- Constraints: ...

【タスク分解（≤2h）】
1. ...
2. ...

【最初のRedテスト提案】
- 追加先: tests/test_....py
- テスト名: test_...
- 失敗の根拠: 未実装/意図的失敗/例外期待

【状態ファイル】
- 作成: .claude/flow/state.json
- 初期phase: red
- current_task: ...
```

## 注意・安全

- **Redテストは必ず失敗**させてください（Greenの過剰実装を防ぐため）
- 既存stateがある場合は **強制上書きしない**（`--reset` 明示時のみ）
- リポジトリが空/未初期化なら必要最小のGit初期化手順を提案
