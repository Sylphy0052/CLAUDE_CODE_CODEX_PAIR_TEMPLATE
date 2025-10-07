---
description: "[実装フェーズ実行] 現在のタスクに対する最小実装を行い、テストを通過させます。--codex指定時は Bash で codex CLI を実行し、出力をファイルに反映します。"
argument_hint: "[--codex (任意)]"
notes: |
  バージョン: 2.3（Codex CLI連携対応）
  対応フェーズ: green
  依存: .claude/.flow/state.json, tasks.md, tests/
---
# Flow: 実装フェーズ単独実行コマンド (implement)

このコマンドは TDD サイクル中の **Green フェーズ（実装段階）** を単独で実行します。  
`flow-next` の中で行われる「テストを通す最小実装」処理を個別に呼び出す用途に対応します。  
`--codex` オプションを指定すると、**Bash経由で codex CLI を呼び出し、生成されたコードや差分をファイルへ反映**します。

---

## 実行フロー

### ステップ1: 状態と前提チェック

- `.claude/.flow/state.json` を読み込み。
  - `phase` が `"green"` でない場合は警告を出して終了。
  - 破損している場合は `/flow-reset` を案内。
- `tests/` ディレクトリ配下に最新テストが存在するか確認。

---

### ステップ2: 実装生成

#### 通常モード（--codex なし）

1. 現在のテスト失敗ログを解析し、未定義関数・未実装ロジックを特定。
2. 対応するファイルに最小限のコードを追加。
3. 差分を生成して `## DIFF` に出力。
4. `pytest -q` 実行 → 全テスト成功を確認。
5. 成功時は `phase` を `"refactor"` に更新。

---

#### Codexモード（--codex あり）

1. **Codexプロンプトを生成**
   - 入力内容には以下を含める:
     - 最新テストの失敗内容
     - 関連関数の定義抜粋
     - 期待される出力形式（udiffまたはファイル本文）
2. **Bashで codex CLI 実行**
   - 実行例:

     - ```
       PROMPT_FILE=$(mktemp)
       OUT_FILE=$(mktemp)
       echo "$CODEX_PROMPT" > "$PROMPT_FILE"
       codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1 || true
       ```

3. **codex 出力を解析・反映**
   - udiff形式 → そのまま差分適用。
   - 本文形式 → ファイルパス検出→本文書込→ローカルdiff生成。
   - 適用後 `## DIFF` に反映内容を記録。
4. **フォーマット・テスト**
   - `black` / `ruff` 実行（ログ追記）。
   - `pytest -q` で成功確認。
5. 成功時:
   - `phase` → `"refactor"` に更新。
   - `## COMMIT` に `feat:` タグ付きメッセージを生成。

---

### ステップ3: 出力セクション

---

## DIFF

<codex生成差分、またはローカル変更のdiff>
---

## TEST_LOG

<pytest結果、black/ruff結果、codex CLI実行ログ>
---

## COMMIT

<Conventional Commits形式 日本語メッセージ>
---

## STATE

<更新後の state.json 全文>
---

---

## エラーハンドリング

| 状況 | 対応 |
|------|------|
| テスト失敗 | ロールバックしてエラーメッセージを表示 |
| codex出力不正 | 適用中止、`TEST_LOG` に「Codex出力不明」記録 |
| 書込権限なし | 処理停止（安全停止） |
| pytest失敗 | stateを変更せず中断、ユーザーに手動修正を促す |

---

## 実装メモ

- 元の実装では `@implementer` と `@git-committer` が協調していましたが、本版ではコマンド内で直接ファイル更新・テスト実行・状態更新を行います。
- Git操作は削除。state.json 書換のみで完結。
- Codexモードでは codex CLI が PATH 上にあることが前提です。
- 差分適用時は `.claude/.flow/state.json` の `history` に適用ログを追加。

---

## 使用例

```
/implement

# 通常の最小実装を実行

/implement --codex

# Bash経由で codex CLI による実装補完を適用

```
