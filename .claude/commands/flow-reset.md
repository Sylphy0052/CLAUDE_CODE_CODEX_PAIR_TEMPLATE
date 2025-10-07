---
description: "[Flowリセット] 現在の .claude/.flow 状態を安全にリセットし、初期状態に戻します。Git操作は行いません。"
argument_hint: "[--hard (任意)]"
notes: |
  バージョン: 2.1（Git非依存版）
  依存: .claude/.flow/state.json, tasks.md
  出力: リセットレポート + state.json 再生成 + tasks.md 再同期
---
# Flow: 状態リセットコマンド (flow-reset)

`flow-reset` は、壊れた `state.json` や `tasks.md` を検出した場合に、**初期状態へ安全に復旧**します。
`--hard` オプションを指定すると、現在のタスク進行状況も破棄して完全初期化します。

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

```
{
  "phase": "red",
  "current_task": "<tasks.md の最初の未完了タスク>",
  "completed_tasks": [],
  "pending_tasks": [],
  "history": []
}
```

- `--hard` でなければ、`completed_tasks` に `[x]` 行の内容を再登録。
- `pending_tasks` は `tasks.md` 内の `[ ]` 行から再構築。

---

### ステップ4: タスクファイルの整合性修復

- `tasks.md` に欠損や異常構文がある場合は再整形。
- `# Tasks` 見出しがない場合は自動付与。
- 正常な行フォーマット（`- [ ] 内容`）に統一。

---

### ステップ5: 結果出力

リセット結果は標準出力に次のように出力します。

```
🔄 Flow state has been reset.
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

---

## 使用例

```
/flow-reset

# state.json の破損を検出して安全に再生成

/flow-reset --hard

# 全タスクと履歴を破棄し完全リセット

```
