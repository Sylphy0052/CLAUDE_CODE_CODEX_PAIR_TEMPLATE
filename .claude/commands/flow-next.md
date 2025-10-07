---
description: "[TDDサイクル実行] 現在フェーズ(red|green|refactor)を検出し、TDD 1ステップ（Red→Green→Refactorのいずれか）を単独で実行します。--codex 指定時は Bash で codex CLI を実行し、出力をファイルへ反映します。"
argument_hint: "[--codex (任意)]"
notes: |
  バージョン: 2.3（Codex連携をBash経由で実行し、結果を確実に反映）
  依存: .claude/.flow/state.json, .claude/.flow/tasks.md, pytest, git
  追加依存(--codex時): codex CLI（PATHに存在すること）
---
# Flow: TDDサイクル実行コマンド (flow-next)

`.claude/.flow/state.json` を読み、現在の `phase`（red / green / refactor）に応じた処理を**1回**だけ実行します。
`--codex` オプションがある場合、**Bash で codex CLI を呼び出し**、生成結果（diff またはコード本文）を**実ファイルへ反映**します。

**前提**: `/flow-init <feature-name>`で仕様書からTDDフローが初期化済みであること。

---

## 実行フロー（共通）

### ステップ0: 前提チェック

- `.claude/.flow/state.json` と `.claude/.flow/tasks.md` を読み込み。なければ `/flow-init` を案内して終了。
- 規模ガード: **変更ファイル ≤ 3、総diff行数 ±120（Codexモード時は ±200）** を超えそうなら中断。

### ステップ1: フェーズ分岐

#### 🟥 Red（失敗するテストの追加）

1. current_task の受入基準から**失敗するテスト**を `tests/` 配下に作成。
2. `pytest -q` を実行し、「失敗」を確認（ログは `## TEST_LOG`）。
3. `phase` を `green` に更新して終了。

> Red フェーズは Codex を使わず、通常どおりに生成・検証します。

---

#### 🟩 Green（テストを通す最小実装）

**通常モード（--codex なし）**

1. 失敗テストを最小修正で通す実装を作成し、ファイルへ書き込み。
2. `pytest -q` で成功確認。成功なら `phase=refactor` に更新。

**Codexモード（--codex あり）**

1. **Codex入力プロンプトの構築**（内部生成・要点）
   - 失敗テスト名/エラーメッセージ、関連ファイル抜粋、期待I/Oや境界条件。
   - 期待出力形式は**可能なら unified diff（udiff）**を優先。
2. **Bashで codex CLI 実行**
   - 実行例（擬似。あなたの環境のcodex引数に合わせて解釈）:

     - ```
       PROMPT_FILE=$(mktemp)
       OUT_FILE=$(mktemp)
       echo "$CODEX_PROMPT" > "$PROMPT_FILE"

       # diffを優先生成。失敗したら本文生成に切替

       codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1 || true
       ```

3. **codex出力の判定 → 反映**
   - **udiff 形式を検出**できた場合: そのまま `## DIFF` に格納し、**差分適用**を実施。
   - **udiffでない場合**（ファイル本文やパッチ擬似表現など）:
     - 出力から**ファイルパスと本文**を抽出し、該当パスに**直接書き込み**。
     - 書き込み結果から**ローカルdiffを生成**して `## DIFF` に格納。
4. **フォーマッタ & テスト**
   - 必要に応じて `black` / `ruff` を実行し、結果を `## TEST_LOG` 末尾に追記。
   - `pytest -q` 実行。**成功**なら `phase=refactor` に更新。失敗なら**全適用をロールバック**して中断。
5. `## COMMIT` に Conventional Commits（日本語）メッセージを生成（例: `feat: 〇〇を実装し tests/test_xxx.py を通過`）。

---

#### 🟦 Refactor（動作不変の品質改善）

**通常モード（--codex なし）**

1. 命名/重複/凝集度などの**安全な最小リファクタ**を適用。
2. `pytest -q` を実行し成功維持を確認。OKなら**タスクを完了**（tasks.md の該当行を `[x]` / state.json の completed に移動、次タスクを current に設定、`phase=red`）。

**Codexモード（--codex あり）**

1. **Codex入力プロンプトの構築**（内部生成・要点）
   - 対象ファイル、改善方針（命名統一・抽象化・型補強など）、**動作不変**であることの強調。
   - 期待出力は**udiff**を優先。
2. **Bashで codex CLI 実行**
   - 実行例:

     - ```
       PROMPT_FILE=$(mktemp)
       OUT_FILE=$(mktemp)
       echo "$CODEX_PROMPT" > "$PROMPT_FILE"
       codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1 || true
       ```

3. **出力の判定 → 反映**
   - udiff を検出 → そのまま差分適用。
   - udiff でない → パス+本文を抽出して**上書き**し、ローカルdiffを生成。
4. **フォーマッタ & テスト**
   - `black` / `ruff` を必要に応じて実行（ログ追記）。
   - `pytest -q` 成功維持を確認。失敗時は**ロールバック**。
5. **タスク完了処理**
   - tasks.md を `[x]` に更新、state.json を completed/pending/current に反映。
   - 次タスクがあれば `phase=red` に、無ければ `phase=red` のまま idle。

---

### ステップ2.5: ロールバック戦略（共通）

- 差分適用や本文上書きの直後に `pytest -q` が**失敗**した場合、**直前のファイルスナップショット**に戻します。
- Codex出力が**空 / パース不能 / 規模上限超過**なら適用せず中断。

---

### ステップ3: 出力セクション（厳守）

---

## DIFF

<適用した unified diff（Codexモード時は必要に応じて `# Codex:` コメント付き）>
---

## TEST_LOG

<pytest の出力と、（該当時のみ）black/ruff の実行結果、codex 実行ログ（コマンド・終了コード・要約）>
---

## COMMIT

<Conventional Commits 日本語メッセージ>
---

## STATE

<更新後の state.json 全文>
---

---

## Codexモードの仕様詳細（--codex）

- **実行形態**: **Bash から `codex` CLI を直接実行**します。プロンプトはテンポラリに保存し、標準出力を成果物として回収します。  
  - 代表的な動作（疑似例）

    - ```

      # Green: 実装生成（udiffを優先）

      codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1

      # Refactor: 改善案（udiffを優先）

      codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1
      ```

- **出力の取り扱い**:
  - **udiff** ならそのまま `## DIFF` に格納し適用。
  - **本文**（ファイル単位）なら、**パス抽出→本文上書き→ローカルdiff生成**の順で反映。
- **ログ**:
  - `## TEST_LOG` に codex 実行の**コマンド要約・終了コード・出力量サイズ**を記録（標準出力は長すぎる場合ダイジェスト化）。
- **規模上限**:
  - 通常 ±120 行 → Codexモード **±200 行**。
- **失敗時**:
  - パース不可・テスト失敗・規模超過は**適用中止**し、**元ファイルへロールバック**します。
  - 失敗理由を `## TEST_LOG` に明示。

---

## 使用例

- 通常:

  - ```
    /flow-next

    # 現在フェーズを自動検出して1ステップ実行

  ```

- Codexモード（CLI連携有効）:

  - ```
    /flow-next --codex

    # Bash経由で codex を実行し、生成結果をファイルに反映

  ```

---

## 実装メモ

- `codex` コマンドの引数・モデル指定は環境に合わせて調整してください（例: `codex --model gpt-xxx --mode diff`）。
- udiff判定は `^diff --git`, `^@@` などのシグネチャで簡易チェック。非udiffの場合は `^// FILE: path` などのガイドコメント基準でパス抽出を試みます。
- **差分適用前**に一時バックアップを取り、失敗時に戻せるようにしておくと安全です。
