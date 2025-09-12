---
allowed-tools: Bash(git:*), Bash(codex:*), Read(*.md), Fetch(*)
description: "タスク単位で Red→Green→Refactor の TDD サイクルを進めます。planned からの初回 Red、Codex 実装/テスト作成、相互レビュー、state/tasks の自動更新に対応。"
updated: "2025-09-12"
---

# flow-next

`.claude/flow/state.json` と `.claude/flow/tasks.json` を基に、**タスク単位**で **TDD サイクル（Red→Green→Refactor）** を 1 回または複数回（`--cycles`）実行します。
初回で `phase=planned` の場合は **最初の Red** を開始します。

- 司令塔: **Claude Code**（オーケストレーション／テスト実行・判定／state & tasks 更新）
- 実装/テスト作成: **Codex**
- レビュー: **Claude Code & Codex の相互レビュー**（不一致時は Claude が最終判断）
- フェーズ管理: `planned → red → green → refactor`（自動判定。明示指定で上書き可）

---

## 事前条件

- `.claude/flow/state.json` が存在し、`current_task` が設定済み（無ければ `flow-init`）
- `.claude/flow/tasks.json` に `current_task` のレコードがあり、`status ∈ {planned, implementing, reviewing, testing}`

---

## 引数

- `--cycles <N>` : サイクル最大回数（既定: 1）
- `--phase <red|green|refactor>` : 単一フェーズ実行（自動判定を上書き）
- `--max-diff <lines>` : 差分表示/適用の上限（既定: 200）
- `--no-codex` : Codex 依存処理を抑止（Claude 単独範囲のみ）
- `--dry-run` : ファイルを変更せず提案のみ
- `--wip-allow` : `.claude/flow/config.json` の `wip_limit` を無視
- `--format <md|json>` : **出力形式**を選択（既定: `md`） ← 追加
- `--compact` : **簡易表示**（主要項目のみ。補足や詳細を省略） ← 追加
- `--verbose` : **詳細表示**（要約に加え、カバレッジ/リンタ/型の詳細や追加ログも出力） ← 追加

---

## 実行フロー（1 サイクルの概要）

1. **前提チェック**：DAG/スキーマ/ブロッカー確認
2. **Red**（`planned|red`）：最小 Red テストを追加 → 失敗確認
3. **Green**：最短実装（unified diff）→ テスト成功確認
4. **Refactor**：外部挙動不変の整理 → 成功維持確認
5. **相互レビュー**：Claude & Codex の所見統合（不一致は Claude が最終判断）
6. **更新**：`state/tasks/docs` を自動更新、アンカー（`docs/tasks.md#t###-slug`）を反映

---

## 出力（デフォルト = Markdown）

【状態要約】

- Phase: `<planned|red|green|refactor>`
- Current Task: `<T###-slug>`
- Executed Cycles: `k / 指定値`
- Next Tasks (Top5): `[T002](docs/tasks.md#t002-...)`, `[T003](docs/tasks.md#t003-...)`
- 直近ログ: `...`

【提案差分（必要時のみ）】

```diff
<unified diff>
```

【テスト結果（要約）】

- ランナー: `passed/failed`（**要約のみ**）
- カバレッジ / リンタ / 型チェック: **要約のみ**
  - 詳細は **`flow-run`** または **`--verbose`** を使用してください。

【レビュー統合（要約）】

- Claude 所見: `...` / Codex 所見: `...` / 統合見解: `OK` or `要修正`

> 差分が無いときは **diff ブロックを表示しません**。冗長な解説は省き、詳細運用は `flow-run` を参照。

---

## 出力（JSON 例 / `--format json`）

```json
{
  "phase": "green",
  "current_task": "T001-login-feature",
  "executed_cycles": 1,
  "next_tasks": [
    {"id": "T002-session-store", "anchor": "docs/tasks.md#t002-session-store"}
  ],
  "last_diff": "PATCH_OR_NULL",
  "test": {
    "result": "passed",
    "summary": {"passed": 12, "failed": 0, "skipped": 1}
  },
  "review": {
    "claude": "OK",
    "codex": "OK",
    "final": "OK"
  },
  "updated_at": "2025-09-12T10:00:00+09:00"
}
```

`--compact` の場合、`next_tasks` は最大3件、`test.summary` などの詳細を省略します。
`--verbose` の場合、`test.coverage_detail` / `lint.issues[]` / `typecheck.issues[]` 等を追加出力します。

---

## 安全策・停止条件

- `--max-diff` 超過 → 停止し「タスク分割」提案
- Red で失敗しない / Green で成功しない → 停止し原因を要約
- 破壊的変更の兆候 → 停止し明示同意を要求
- 依存 DAG 不整合（循環/孤立）→ 停止し `warnings` に詳細を出力

---
