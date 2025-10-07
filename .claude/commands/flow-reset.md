---
description: "[Flowリセット] 現在の .claude/.flow 状態を安全にリセットし、初期状態に戻します。Git操作は行いません。"
argument_hint: "[--hard (任意)]"
notes: |
  バージョン: 2.2（SDD対応版）
  依存: .claude/.flow/state.json, tasks.md
  出力: リセットレポート + state.json 再生成 + tasks.md 再同期
---
# Flow: 状態リセットコマンド (flow-reset)

`flow-reset` は、壊れた `state.json` や `tasks.md` を検出した場合に、**初期状態へ安全に復旧**します。
`--hard` オプションを指定すると、現在のタスク進行状況も破棄して完全初期化します。

**注意**: 仕様書（`.claude/specs/`）は削除されません。実装状態のみリセットします。

---

## 実行フロー

### ステップ1: 状態確認

- `.claude/.flow/state.json` の存在確認。
  - 存在しない場合 → `/flow-init` の実行を案内して終了。
- `state.json` のパース結果が不正／欠損／破損している場合:
  - バックアップ（`.claude/.flow/state.json.bak`）を作成。
  - 再生成対象としてマーク。

---

### ステップ2: タスク状態の解析とリセット内容決定

- `tasks.md` をスキャンして、`[x]`（完了）と `[ ]`（未完了）を区別。
- `--hard` 指定なし:
  - 完了済みタスクは維持。
  - 現在フェーズ (`phase`) は `"red"` にリセット。
  - 現在タスク (`current_task`) は次の未完了タスクへ更新。
- `--hard` 指定あり:
  - 全タスクを未完了状態に戻す。
  - `phase` は `"red"` にリセット。
  - `completed_tasks` / `history` は空に初期化。

---

### ステップ3: state.json の再生成

```json
{
  "phase": "red",
  "current_task": "<tasks.md の最初の未完了タスク>",
  "completed_tasks": [],
  "pending_tasks": [],
  "history": [],
  "spec": {
    "feature_name": "[元の機能名]",
    "spec_path": "[元の仕様書パス]",
    "has_design": true,
    "has_requirements": true
  }
}
```

- `--hard` でなければ、`completed_tasks` に `[x]` 行の内容を再登録。
- `pending_tasks` は `tasks.md` 内の `[ ]` 行から再構築。
- `spec` フィールドは元の値を保持（仕様書との連携を維持）。

---

### ステップ4: タスクファイルの整合性修復

- `tasks.md` に欠損や異常構文がある場合は再整形。
- `# Tasks` 見出しがない場合は自動付与。
- 正常な行フォーマット（`- [ ] 内容`）に統一。

---

### ステップ5: 結果出力

リセット結果は標準出力に次のように出力します。

```text
🔄 Flow state has been reset.

📦 仕様書: [feature-name]（連携維持）
Phase: red
Current Task: <再設定されたタスク>
Completed Tasks: N件
Pending Tasks: M件

Next Step: /flow-next でサイクルを再開してください。
```

---

## オプション

| オプション | 内容 |
|-------------|------|
| `--hard` | タスク進捗も含め完全初期化します。バックアップは自動生成されます。 |

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| state.json 不在 | `/flow-init` を案内 |
| state.json パースエラー | `.bak` に退避し再生成 |
| tasks.md 不整合 | 自動修正（`- [ ]` 形式に正規化） |
| 書込権限なし | 標準エラーに通知して終了 |

---

## 実装メモ

- 元の `@state-manager` 呼び出し（状態リセット担当）を削除し、自己完結処理に置換。
- `state.json` の再構築は flow-init と同一スキーマを使用。
- Git連携は削除し、ローカルファイル操作のみで完結。
- `--hard` は破壊的操作のため、処理前に `.bak` バックアップを常に生成。
- **SDD対応**: `spec` フィールドを保持し、仕様書との連携を維持（`.claude/specs/`は影響を受けない）。

---

## 使用例

```bash
/flow-reset

# state.json の破損を検出して安全に再生成
# 仕様書連携は維持される

/flow-reset --hard

# 全タスクと履歴を破棄し完全リセット
# 仕様書連携は維持される
```
