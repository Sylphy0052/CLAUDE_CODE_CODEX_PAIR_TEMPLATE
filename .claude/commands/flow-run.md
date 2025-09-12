---
allowed-tools: Bash(git:*), Bash(codex:*), Read(*.md), Fetch(*)
description: "指定したマイルストーンまたはフェーズをまとめて進め、最後に総合テスト・相互レビュー・人間承認を行います。非対話モード対応 (--yes/--no-input)、--verbose で詳細ログ出力。"
updated: "2025-09-12"
---

# flow-run

`.claude/flow/state.json` / `.claude/flow/tasks.json` を読み、**1マイルストーン**または**1フェーズ**を連続実行します。
内部では `flow-next --cycles 1` を繰り返し、完了後に **総合テスト → 相互レビュー（Claude & Codex） → 人間承認** を行います。

---

## 実行モード（いずれか必須）

- `--milestone "<name|index|current>"` : 指定マイルストーンが完了条件を満たすまで反復
- `--phase <red|green|refactor>` : 指定フェーズを完了条件まで反復

---

## 制御オプション

- `--max-steps <M>` : 内部反復上限（既定: 10）
- `--no-codex` : Codex 呼び出し抑止（Claude 単独範囲のみ）
- `--dry-run` : 適用せずに結果のみ表示
- `--confirm` : 適用直前に承認確認（既定）
- `--yes` : **全承認を自動化**（承認プロンプトを表示せず Yes 扱い） ← 追加
- `--no-input` : **全プロンプトを抑止**（CI 用。承認確認も含め無入力モード） ← 追加
- `--format <md|json>` : 出力形式（既定: `md`）
- `--compact` : 簡易表示（主要項目のみ）
- `--verbose` : **詳細表示**（テスト/カバレッジ/リンタ/型チェックの詳細ログを追加） ← 追加

---

## 処理の流れ

1. **対象確定**
   - `--milestone` or `--phase` を解析し終了条件を決定

2. **まとめ実行**
   - `flow-next --cycles 1` を必要回数呼び出し
   - 安全策（`--max-diff` 超過 / DAG不整合 / Red失敗しない等）に該当したら中断

3. **総合テスト（要約）**
   - tests / coverage / lint / typecheck の **要約のみ**を表示
   - 詳細が必要な場合は **`--verbose`** を使用

4. **レビュー統合**
   - Claude と Codex の所見を照合し統合見解を作成
   - 不一致時は Claude が最終判断（理由付き）

5. **人間承認 → 適用**
   - **承認プロンプトはここで1回だけ表示**（`--yes` or `--no-input` 指定時はスキップ） ← 修正
   - Yes → 適用・履歴更新・`CHANGELOG` 案を生成
   - No → 変更破棄（ロールバック案を提示）

---

## 出力（Markdown, 既定）

【実行サマリ】

- mode: `milestone|phase`
- target: `<milestone-name|index|phase>`
- executed_steps: `K / max-steps`
- stop_reason: `<completed|safety-limit|error|user-stop>`

【差分要約（存在時のみ）】

```diff
<unified diff>
```

【総合テスト（要約）】

- tests: `passed=X / failed=Y / skipped=Z`
- coverage: `lines=..% / branches=..%`
- lint: `issues=N`
- typecheck: `errors=M`

> 詳細は `--verbose` を使用

【レビュー要約】

- Claude: `...`
- Codex: `...`
- 統合見解: `OK` / `要修正（理由）`

【承認】

- 適用しますか？ (Yes/No)
  - `--yes` または `--no-input` 指定時は **この確認をスキップ** ← 修正

【リリースノート案】

- 機能: ...
- 修正: ...
- 既知の問題: ...

---

## 出力（JSON, `--format json`）

```json
{
  "mode": "milestone",
  "target": "M1 Authentication",
  "executed_steps": 4,
  "stop_reason": "completed",
  "diff": "PATCH_OR_NULL",
  "tests": {
    "summary": {"passed": 120, "failed": 0, "skipped": 3},
    "coverage": {"lines": 87.4, "branches": 79.2},
    "lint": {"issues": 0},
    "typecheck": {"errors": 0},
    "detail_included": false
  },
  "review": {
    "claude": "OK",
    "codex": "OK",
    "final": "OK"
  },
  "approval_required": true,
  "approved_auto": true,
  "release_notes": {
    "features": ["..."],
    "fixes": ["..."],
    "known_issues": []
  },
  "updated_at": "2025-09-12T10:00:00+09:00"
}
```

- `approval_required`: 常に true
- `approved_auto`: `--yes` or `--no-input` 使用時に true

---

## 安全策・停止条件

- `--max-steps` 超過 → 停止（要約提示）
- 大規模差分 → 停止し「分割提案」
- DAG不整合 → 停止し `warnings` に出力
- Red が失敗しない / Green が成功しない → 停止して原因要約
- 破壊的変更の兆候 → 停止し明示同意を要求

---

## 参照

- `flow-init` : 初期化（phase=planned）
- `flow-status` : 現在地要約
- `flow-next` : 単発の TDD サイクル実行
