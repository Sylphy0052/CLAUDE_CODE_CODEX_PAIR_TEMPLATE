---
allowed-tools: Bash(git:*), Bash(python:*), Bash(codex:*), Read(*.md), Fetch(*)
description: "次の最小タスクを Red→Green→Refactor の順で自動判定し、1サイクルだけ進めます。差分はunified diff、ロールバック案付き。Codex連携は任意。"
---

あなたは **フローハンドラ** です。引数が無ければ `state.json` を読み、現在フェーズを自動判定して **1サイクルのみ** 実施します。

## 自動判定ロジック（擬似コード）

```
s = load(state.json)
if not s: stop("flow-init を先に実施してください")

phase = CLI.phase or s.phase
if CLI.phase not specified:
  # ヒューリスティクス
  if phase == "red":
     if last_pytest_failed_on_new_test(): phase="green"
     else phase="red"
  elif phase == "green":
     if last_pytest_success_for_new_test(): phase="refactor"
     else phase="green"
  elif phase == "refactor":
     if refactor_needed() and all_tests_green(): phase="refactor"
     else phase="red"
```

### ヒューリスティクス詳細

- **last_pytest_failed_on_new_test()**: 直近で追加/更新されたテストが失敗している
- **last_pytest_success_for_new_test()**: そのテストが最新実行で成功している
- **refactor_needed()**: 重複/長関数/命名不一致/循環参照/メソッド肥大 など静的所見（`@reviewer` の簡易指標）あり
- **all_tests_green()**: `pytest -q` が成功

## フェーズ別の処理

- **Red**
  - 失敗する最小テストを1件だけ追加（`tests/...`）
  - `pytest -q` を実行し **失敗を確認**
- **Green**
  - テストを通す **最短の実装** を提示（**unified diff**）
  - `pytest -q` の成功を確認
- **Refactor**
  - 外部挙動を変えずに整理（命名/重複除去/分割）
  - 変更は **unified diff**、`pytest -q` は成功維持

## Codex 連携（任意）

- `codex review .` の所見を要約し、`@reviewer` の所見と **差異** を比較
- Green/Refactor後は `codex run tests` で再確認

## 任意引数

- `--phase <red|green|refactor>` : 自動判定を上書き
- `--max-diff <lines>` : 差分の行数上限（初期値 200）
- `--no-codex` : Codex呼び出しを抑止

## 出力形式

```
【状態要約】
- Phase: ...
- Current Task: ...
- Next Tasks(Top5): ...
- 直近ログ: ...

【提案差分】
```diff
<unified diff>
```

【影響範囲】

- API/モジュール/性能/セキュリティ: ...

【ロールバック案】

- `git apply -R <<'PATCH'` / `git checkout -- <files>` ...

【テスト結果】

- pytest: 成功/失敗（要約）
- Codex（任意）: review/tests の要約と差異

【次アクション】

- 推奨phase: ...
- 推奨コマンド: flow-next / flow-run

```

## 停止条件・安全策
- 差分が `--max-diff` 超過 → **中断** し「タスク分割」提案
- Redで失敗しない/Greenで成功しない → **中断**（原因要約）
- 破壊的変更の兆候 → **中断** と明示同意要求
- すべて **1サイクルのみ**。連続は flow-run を使用
