---
description: "[Flowステータス表示] 現在のフェーズ・タスク・進捗状況を一覧表示します。"
argument_hint: ""
notes: |
  バージョン: 2.1（Git非依存 / サブエージェント削除）
  依存: .claude/.flow/state.json, tasks.md
  出力: 状態レポート + 進捗統計
---
# Flow: ステータス確認コマンド (flow-status)

Flow管理の現在状態を可視化するためのコマンドです。  
TDDサイクル中のフェーズ（red / green / refactor）や、タスクの進行状況を一覧化します。  
`flow-init`・`flow-next`・`flow-run` などのコマンド実行後に使用して、進捗を確認します。

---

## 実行フロー

### ステップ1: 状態ファイルの読込

- `.claude/.flow/state.json` を読み込み、以下のキーを取得します:
  - `phase`
  - `current_task`
  - `completed_tasks`
  - `pending_tasks`
  - `history`
  - `spec` (SDDモードの場合)
- `.claude/.flow/tasks.md` もあれば同時に読み込み、最新状態を同期。

### ステップ1.5: 仕様情報の確認（SDDモード時のみ）

`state.json`に`spec`フィールドが存在する場合、仕様書情報を取得・確認：

- `spec.feature_name`: 機能名
- `spec.spec_path`: 仕様書パス（通常は`.claude/specs/[feature-name]/`）
- `spec.has_design`: 設計書の有無
- `spec.has_requirements`: 要件定義の有無

仕様書ファイルの存在確認：

- `.claude/specs/[feature-name]/spec.json`
- `.claude/specs/[feature-name]/design.md`（has_design=trueの場合）
- `.claude/specs/[feature-name]/requirements.md`（has_requirements=trueの場合）

### ステップ2: 進捗集計

- `tasks.md` の行を走査して集計:
  - `[x]` → 完了タスク
  - `[ ]` → 未完了タスク
- 完了率を算出:
  - `completion = round(completed / total * 100, 1)`
- `phase` に応じて次のアクションを推定:
  - `"red"` → 「新しいテストを作成する」
  - `"green"` → 「テストを通す実装を行う」
  - `"refactor"` → 「コードを安全に改善する」

---

## 出力フォーマット

### パターンA: SDDモード（仕様書連携あり）

```text
📊 Flow Status
─────────────────────────────
📦 Spec: user-authentication
   Path: .claude/specs/user-authentication/
   - spec.json ✅
   - design.md ✅
   - requirements.md ✅

Phase          : red
Current Task   : ログイン処理のテストケースを作成
Progress       : 3 / 7 (42.8%)
Next Action    : 新しいテストを作成してください

✅ Completed Tasks:

- ユーザー登録フォームのテスト作成
- バリデーションロジックの実装
- 登録テスト通過確認

🕓 Pending Tasks:

- [ ] ログイン処理のテストケースを作成
- [ ] セッション維持のテスト
─────────────────────────────
Last Updated: 2025-10-03 15:12:44
```

### パターンB: TDDモード（仕様書なし）

```text
📊 Flow Status
─────────────────────────────
Phase          : red
Current Task   : ログイン処理のテストケースを作成
Progress       : 3 / 7 (42.8%)
Next Action    : 新しいテストを作成してください

✅ Completed Tasks:

- ユーザー登録フォームのテスト作成
- バリデーションロジックの実装
- 登録テスト通過確認

🕓 Pending Tasks:

- [ ] ログイン処理のテストケースを作成
- [ ] セッション維持のテスト
─────────────────────────────
Last Updated: 2025-10-03 15:12:44
```

---

### ステップ3: 補助出力

オプションで、詳細ビューとサマリビューを切り替え可能。

| オプション | 内容 |
|-------------|------|
| `--summary` | 完了率と次アクションのみ出力（簡易ビュー） |
| `--all`     | 完了履歴 (`history`) を全て表示（デフォルトは最新5件） |

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| `state.json` 不存在 | `/flow-init` の実行を促すメッセージを表示 |
| JSON破損 | `.bak` に退避して終了 |
| tasks.md 不存在 | 進捗率を算出せず、状態情報のみ表示 |
| 仕様書ファイル不存在（SDDモード時） | 警告を表示するが処理は継続 |

---

## 実装メモ

- 元の実装では `@state-manager` サブエージェントが状態読込と集計を行っていたが、本版では直接JSONを解析。
- 出力はターミナル互換のテキスト整形で表示（表罫線と絵文字を用いた視認性向上）。
- Gitや外部ツールとの連携は削除。完全ローカル動作。
- SDDモード時は、`state.json`の`spec`フィールドから仕様書情報を取得し、仕様書ファイルの存在確認を行う。
- 仕様書情報は表示されるが、存在確認で問題があっても処理は継続する（読み取り専用のため）。

---

## 使用例

```bash
/flow-status

# 詳細表示（デフォルト）
# SDDモードの場合は仕様書情報も表示

/flow-status --summary

# 簡易表示（進捗率と現在フェーズのみ）

/flow-status --all

# 完了履歴をすべて表示
```
