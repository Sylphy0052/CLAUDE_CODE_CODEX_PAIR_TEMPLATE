---
description: "[Flow初期化] .claude/.flow ディレクトリを作成し、state.json と tasks.md を初期化します。Git操作は一切行いません。"
argument_hint: "[--reset (任意)]"
notes: |
  バージョン: 2.2（Git連携削除版）
  依存: 現在の作業ディレクトリ
  出力: 初期化レポート + state.json + tasks.md
---
# Flow: プロジェクト初期化コマンド (flow-init)

`.claude/.flow/` ディレクトリを新規作成し、Flow管理用の初期ファイルを生成します。
Git連携は削除され、**完全にローカルファイル操作のみ**で動作します。

---

## 実行フロー

### ステップ1: 事前チェック

- `.claude/.flow/` の存在確認:
  - 存在しない場合 → 新規作成。
  - 存在する場合:
    - `--reset` 指定あり → 既存ファイルを `.bak` にバックアップ後、再生成。
    - `--reset` 指定なし → 警告を表示して終了。

---

### ステップ2: 初期ファイル生成

#### state.json

```
{
  "phase": "red",
  "current_task": "初期タスク",
  "completed_tasks": [],
  "pending_tasks": [],
  "history": []
}
```

- `phase`: `"red"` で固定。
- `current_task`: `"初期タスク"`。
- 今後の `/flow-next`, `/flow-status` で参照・更新されます。

#### tasks.md

```

# Tasks

- [ ] 初期タスク: コア機能のテストケースを作成する
```

- `tasks.md` はタスクリスト管理用。
- Flow実行時に `[ ]` が `[x]` に変化します。

---

### ステップ3: 結果出力

初期化が成功すると次の出力を返します。

```
✅ Flow initialized successfully.
Phase: red
Current Task: 初期タスク
Next Step: /flow-next で最初のテストを生成してください。
```

---

## オプション

| オプション | 内容 |
|-------------|------|
| `--reset` | `.claude/.flow` をバックアップして再初期化 |

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| `.claude/.flow/` 既存・--resetなし | 初期化中止、警告出力 |
| ファイル書込不可 | エラー終了 |
| JSON破損・不整合 | `/flow-reset` の実行を案内 |

---

## 実装メモ

- 元の `@state-manager`・`@planner` 呼び出しは削除済み。
- Git連携 (`git add/commit`) は完全に除外。
- 全出力は標準出力ベースで行われ、外部コマンドに依存しません。
- `flow-init` の結果は `.claude/.flow/state.json` および `tasks.md` に反映され、他の Flow 系コマンドの前提になります。

---

## 使用例

```
/flow-init

# 初回セットアップ

/flow-init --reset

# 再構築（既存データをバックアップ）

```
