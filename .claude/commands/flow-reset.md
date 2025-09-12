---
allowed-tools: Bash(git:*), Read(*.md), Fetch(*)
description: "作業ツリーと state を安全に巻き戻します。未コミット変更は退避し、必要なら直前のバックアップへ復元します。--format/--compact/--dry-run に対応。"
updated: "2025-09-12"
---

# flow-reset

`.claude/flow/state.json` と作業ツリーを **安全に巻き戻す** コマンドです。
> **Codex 連携は行いません（安全性優先）**

---

## 手順

1. **事前確認**
   - 現在のブランチ・未コミットファイル・直近 state を確認
   - 大きな差分がある場合は停止して要約提示

2. **退避（バックアップ）**
   - 未コミット変更を `./.claude/backups/<timestamp>/worktree.patch` に保存
   - `.claude/flow/state.json` を `state.json.bak-<timestamp>` に保存

3. **復元**
   - 既定は直前の state バックアップを復元
   - `--from <path>` 指定があればその state を復元
   - ワークツリーは `git restore --staged --worktree .` 等でクリーンに

4. **検証**
   - `flow-status` を実行し `phase / current_task / next_tasks` を表示
   - 差分が残っていれば再度退避→復元を試行

---

## 引数

- `--hard` : 退避後に**強制的に**作業ツリーをクリーン化（ローカル変更を除外）
- `--from <path>` : 指定したバックアップ (`state.json.bak-*`) から復元
- `--keep-worktree` : ワークツリーを維持し **state のみ**復元
- `--confirm` : 確認プロンプトを表示（既定）
- `--yes` : **全承認を自動化**（確認を出さず実行）
- `--no-input` : **非対話モード**（CI用。全て自動実行）
- `--format <md|json>` : 出力形式（既定: `md`） ← 追加
- `--compact` : 簡易表示（主要項目のみ） ← 追加
- `--dry-run` : **ドライラン**。バックアップ候補・復元対象・差分要約を表示し、変更は行わない ← 追加

---

## 出力形式（Markdown, 既定）

【実行サマリ】

- action: `backup-and-restore | restore-only | noop`
- state_restore_from: `<path or latest>`
- worktree_backup: `<path or none>`
- warnings: [...]

【flow-status（復元後）】

- phase / current_task / next_tasks
- last_diff（要約）
- updated_at

---

## 出力形式（JSON, `--format json`）

```json
{
  "action": "backup-and-restore",
  "state_restore_from": "state.json.bak-20250912-1000",
  "worktree_backup": ".claude/backups/20250912-1000/worktree.patch",
  "warnings": [],
  "restored_status": {
    "phase": "planned",
    "current_task": "T001-login-feature",
    "next_tasks": ["T002-session-store", "T003-rate-limit"],
    "last_diff": null,
    "updated_at": "2025-09-12T10:05:00+09:00"
  }
}
```

`--compact` の場合は `action` / `state_restore_from` / `phase` / `current_task` のみを返します。
`--dry-run` の場合は `action="dry-run"` とし、候補情報のみを出力します。

---

## 動作ルール

- **未コミット差分が巨大** → 停止し「まずコミット or 手動退避」を促す
- **バックアップ作成に失敗** → 停止し理由を表示
- **`--from` が無効** → 停止し利用可能な一覧を提示
- `--yes` または `--no-input` → 確認プロンプトをスキップ

---

## 注意事項

- リセットは**破壊的操作**になり得ます。バックアップが取れない場合は中止してください。
- 復元後は必ず `flow-status` で状態を確認してください。
