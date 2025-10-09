---
description: "[全自動TDD実行] 全タスク完了まで flow-next を自動反復し、完了後に自動コミットまで実行します。ユーザー承認なしで完全自動実行。"
argument_hint: "[--codex (任意)]"
notes: |
  バージョン: 2.0（完全自動実行・放置前提設計・超詳細版）
  依存: .claude/.flow/state.json, .claude/.flow/tasks.md, flow-next.md, ccommit.md
  前提: /flow-init で仕様書からTDDフローが初期化済みであること
---
# Flow: 全自動TDD実行&コミットコマンド (flow-complete) - 超詳細版

## 概要

このコマンドは、**全タスクが完了するまで flow-next を自動反復**し、完了後に **ccommit で自動コミット**を実行します。ユーザーの承認は一切求めず、完全に放置して動作する設計です。

**設計思想**:

- **完全自動実行**: ユーザー承認を一切求めない（放置前提）
- **エラー耐性**: 個別タスクのエラーで全体を停止させない
- **透明性**: 各ステップの進捗を詳細に報告
- **ツール直接使用**: Read/Write/Edit/Bashツールを直接実行（サブエージェント不使用）
- **機能単位コミット**: spec.jsonの機能名を基準に自動コミット

**前提条件**:

- `/flow-init <feature-name>`で仕様書からTDDフローが初期化済み
- `.claude/.flow/state.json` が存在
- `.claude/.flow/tasks.md` が存在
- `.claude/specs/[feature]/spec.json` が存在（仕様書モード時）
- `.claude/specs/[feature]/design.md` が存在（仕様書モード時）

**オプション**:

- `--codex`: 全実装ステップでCodex連携を有効化

---

## ツール使用の原則

### ツール選択基準

1. **Readツール**: ファイルの読み込み（複数ファイルは並列実行）
2. **Writeツール**: 新規ファイルの作成（既存ファイルには使用禁止）
3. **Editツール**: 既存ファイルの部分修正（JSON更新等）
4. **Bashツール**: コマンド実行（git, pytest, poetry, codexなど）

### 並列実行の原則

- **複数ファイルの読み込み**: 依存関係がない場合は並列実行すること
- **独立したBashコマンド**: 依存関係がない場合は並列実行すること
- **依存関係がある操作**: 必ず順次実行すること（例: Write後のBash、Read後の分析）

---

## 実行フロー

### Step 0: 事前チェックと初期化

このステップでは、実行環境の確認と引数の解析を行います。

#### ステップ0-1: コマンド定義ファイルの読み込み（並列実行）

**必須操作**: Readツールを並列で使って以下のファイルを確認

```
複数のReadツールを1つのメッセージで実行:
- .claude/commands/flow-next.md
- .claude/commands/ccommit.md
```

**目的**: 各コマンドの最新仕様を理解し、正確に実行する

**進捗報告**:
```
📖 Step 0-1: コマンド定義ファイルを読み込み中...
✅ 2個のコマンド定義を読み込み完了
```

---

#### ステップ0-2: 引数の解析

**処理手順**:

```python
# 引数取得
ARGS = "$*"  # 全引数を取得
CODEX_FLAG = ""
CODEX_ENABLED = False

# --codexオプションの検出
if "--codex" in ARGS:
    CODEX_FLAG = "--codex"
    CODEX_ENABLED = True
```

**進捗報告**:
```
📋 Step 0-2: 引数解析完了
   Codex連携: [✅ 有効 | ❌ 無効]
```

---

#### ステップ0-3: 前提条件チェック（並列読み込み）

**並列でReadツール実行**:

```python
# 必須ファイルを並列で読み込み
required_files = [
    ".claude/.flow/state.json",
    ".claude/.flow/tasks.md"
]
# Read: required_files を並列読み込み

# state.jsonの検証
if not state_json_exists:
    print("❌ エラー: .claude/.flow/state.json が存在しません")
    print("💡 解決方法: /flow-init <feature-name> を先に実行してください")
    exit(1)

# tasks.mdの検証
if not tasks_md_exists:
    print("❌ エラー: .claude/.flow/tasks.md が存在しません")
    print("💡 解決方法: /flow-init <feature-name> を先に実行してください")
    exit(1)

# state.jsonの解析
state = json.loads(state_json_content)
FEATURE_NAME = state.get("feature_name", "unknown")
SOURCE_SPEC = state.get("source_spec", None)

# タスク数の確認
pending_tasks = state["tasks"]["pending"]
completed_tasks = state["tasks"]["completed"]
total_tasks = len(pending_tasks) + len(completed_tasks)
```

**仕様書モードの確認（条件付き並列読み込み）**:

```python
# source_specが存在する場合、仕様書を読み込み
if SOURCE_SPEC:
    SPEC_MODE = True
    SPEC_DIR = os.path.dirname(SOURCE_SPEC)

    # 仕様書ファイルを並列で読み込み
    spec_files = [
        SOURCE_SPEC,  # spec.json
        f"{SPEC_DIR}/design.md"
    ]
    # Read: spec_files を並列読み込み

    # 存在確認
    if not spec_json_exists:
        print(f"⚠️  警告: 仕様書が見つかりません: {SOURCE_SPEC}")
        SPEC_MODE = False
else:
    SPEC_MODE = False
```

**進捗報告**:
```
✅ Step 0-3: 前提条件チェック完了
   機能: {FEATURE_NAME}
   全タスク: {total_tasks}個
   完了済み: {len(completed_tasks)}個
   残タスク: {len(pending_tasks)}個
   仕様書モード: [✅ 有効 | ❌ 無効]
```

---

#### ステップ0-4: 実行ログの初期化

**Writeツールで新規ログファイルを作成**:

```python
from datetime import datetime

LOG_DIR = ".claude/.logs"
TIMESTAMP = datetime.now().strftime("%Y%m%d_%H%M%S")
LOG_FILE = f"{LOG_DIR}/flow-complete_{TIMESTAMP}.log"

# Bashツールでディレクトリ作成
# mkdir -p .claude/.logs

# Writeツールでログファイル作成
log_content = f"""# Flow-Complete 実行ログ
実行開始: {datetime.now().isoformat()}
機能: {FEATURE_NAME}
全タスク: {total_tasks}個
完了済み: {len(completed_tasks)}個
残タスク: {len(pending_tasks)}個
Codex連携: {CODEX_ENABLED}

────────────────────────────────────────────────────────
"""
# Write: LOG_FILE, log_content
```

**ヘルパー関数の定義**（メモリ内）:

```python
def log_step(step_name, message):
    """ステップ情報をログに追記"""
    log_entry = f"""
[{datetime.now().isoformat()}] {step_name}
{message}
"""
    # Editツールでログファイルに追記

def log_error(step, error_msg):
    """エラー情報をログに記録して終了"""
    error_entry = f"""
❌ エラー発生
ステップ: {step}
エラー: {error_msg}
終了時刻: {datetime.now().isoformat()}
"""
    # Editツールでログファイルに追記
    print(f"\n❌ エラーが発生しました")
    print(f"ステップ: {step}")
    print(f"エラー: {error_msg}")
    print(f"\n📝 詳細ログ: {LOG_FILE}")
    exit(1)

def log_warning(step, warning_msg):
    """警告情報をログに記録（継続）"""
    warning_entry = f"""
⚠️  警告
ステップ: {step}
警告: {warning_msg}
"""
    # Editツールでログファイルに追記
    print(f"\n⚠️  警告")
    print(f"ステップ: {step}")
    print(f"警告: {warning_msg}\n")
```

**進捗報告**:
```
📝 Step 0-4: 実行ログ初期化完了
   ログファイル: {LOG_FILE}
```

---

#### ステップ0-5: 実行開始の宣言

**推定実行時間の算出**:

```python
remaining_tasks = len(pending_tasks)
estimated_minutes = remaining_tasks * 3  # 1タスク平均3分（Red+Green+Refactor）

if CODEX_ENABLED:
    estimated_minutes *= 1.5  # Codexモードは1.5倍

hours = estimated_minutes // 60
minutes = estimated_minutes % 60
```

**進捗報告**:
```
╔════════════════════════════════════════════════════════════╗
║  🚀 flow-complete を開始します                             ║
╚════════════════════════════════════════════════════════════╝

【実行情報】
  機能: {FEATURE_NAME}
  全タスク: {total_tasks}個
  完了済み: {len(completed_tasks)}個
  残タスク: {len(pending_tasks)}個
  Codex連携: [✅ 有効 | ❌ 無効]
  仕様書モード: [✅ 有効 | ❌ 無効]

【実行ステップ】
  Step 1: 全タスク自動実装（Red→Green→Refactorループ）
  Step 2: 最終品質チェック（pytest/mypy/flake8）
  Step 3: 自動コミット（Conventional Commits）
  Step 4: 最終報告

【推定実行時間】
  約 {hours}時間{minutes}分

【開始時刻】
  {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}

════════════════════════════════════════════════════════════
```

---

### Step 1: 全タスク自動実装

このステップでは、全タスクが完了するまで flow-next を自動反復します。

#### ステップ1-0: 初期化

```python
step_count = 0
max_steps = 100  # 安全装置
skipped_tasks = []  # スキップしたタスクの記録
start_time = datetime.now()
```

---

#### ステップ1-1: メインループ

```python
while True:
    # 安全装置チェック
    if step_count >= max_steps:
        log_warning("Step 1", f"最大ステップ数{max_steps}に到達しました。処理を終了します。")
        break

    step_count += 1

    # 状態読み込み
    # Read: ".claude/.flow/state.json"
    state = json.loads(state_json_content)

    current_phase = state["phase"]
    current_task_id = state["current_task"]
    pending_tasks = state["tasks"]["pending"]
    completed_tasks = state["tasks"]["completed"]

    # 全タスク完了チェック
    if len(pending_tasks) == 0:
        print(f"\n✅ 全タスク完了！")
        log_step("Step 1", f"全タスク完了（{step_count}ステップで完了）")
        break

    # タスク情報取得
    # Read: ".claude/.flow/tasks.md"
    current_task = parse_task_from_markdown(tasks_md_content, current_task_id)

    # 進捗表示
    total_tasks = len(pending_tasks) + len(completed_tasks)
    completed_count = len(completed_tasks)
    progress_percentage = (completed_count / total_tasks) * 100

    print(f"\n{'='*60}")
    print(f"[ステップ {step_count}] タスク {completed_count + 1}/{total_tasks} ({progress_percentage:.1f}%)")
    print(f"タイトル: {current_task['title']}")
    print(f"Phase: {current_phase}")
    print(f"{'='*60}")

    # フェーズ別処理
    try:
        if current_phase == "red":
            execute_red_phase(current_task, CODEX_ENABLED)
            # フェーズをgreenに更新
            update_state_phase(".claude/.flow/state.json", "green")

        elif current_phase == "green":
            execute_green_phase(current_task, CODEX_ENABLED)
            # フェーズをrefactorに更新
            update_state_phase(".claude/.flow/state.json", "refactor")

        elif current_phase == "refactor":
            execute_refactor_phase(current_task, CODEX_ENABLED)
            # タスク完了処理
            complete_current_task(".claude/.flow/state.json", current_task_id)
            print(f"✅ タスク完了 (残り {len(pending_tasks) - 1}個)")

    except TaskSkippableError as e:
        # スキップ可能なエラー（テスト恒久失敗、差分規模超過など）
        log_warning("Step 1", f"タスク{current_task_id}をスキップ: {e}")
        skipped_tasks.append({
            "task_id": current_task_id,
            "title": current_task['title'],
            "reason": str(e)
        })
        # タスクをスキップ扱いで完了
        skip_current_task(".claude/.flow/state.json", current_task_id)
        print(f"⚠️  タスクスキップ (残り {len(pending_tasks) - 1}個)")

    except StateInconsistencyError as e:
        # 状態不整合エラー（全体停止）
        log_error("Step 1", f"状態不整合エラー: {e}")
```

---

#### ステップ1-2: Red フェーズの実装

```python
def execute_red_phase(task, codex_enabled):
    """Red フェーズ: 失敗するテストの作成"""

    print(f"  🔴 Red → テスト作成中...")

    # 受入基準からテストケースを生成
    test_file_path = determine_test_file_path(task)
    test_code = generate_failing_test(task["acceptance_criteria"])

    # テストファイルが既存かチェック
    # Bashツール: ls {test_file_path} 2>/dev/null

    if test_file_exists:
        # Readツールで既存テストを読み込み
        # Read: test_file_path

        # Editツールで追記
        # 適切な挿入位置を特定（通常はファイル末尾のクラス内）
        insertion_point = find_test_insertion_point(existing_test_content)

        # Edit: test_file_path,
        #       old_string=insertion_point,
        #       new_string=f"{insertion_point}\n\n{test_code}"
    else:
        # Writeツールで新規作成
        # テストファイルのテンプレート生成
        test_file_content = generate_test_file_template(task, test_code)
        # Write: test_file_path, test_file_content

    # テスト実行（失敗確認）
    # Bashツール: poetry run pytest {test_file_path} -q

    if test_exit_code == 0:
        # テストが成功してしまった場合（想定外）
        raise StateInconsistencyError(f"Redフェーズでテストが成功しました: {test_file_path}")

    print(f"  ✅ テスト失敗を確認")
    log_step("Red Phase", f"タスク{task['id']}: テスト作成完了 ({test_file_path})")


def determine_test_file_path(task):
    """タスクからテストファイルパスを決定"""
    # タスクの実装ファイルパスから推定
    # 例: src/auth/login.py → tests/test_auth/test_login.py
    impl_path = task.get("impl_file", "unknown")
    test_path = convert_impl_path_to_test_path(impl_path)
    return test_path


def generate_failing_test(acceptance_criteria):
    """受入基準から失敗するテストコードを生成"""
    test_cases = []

    for criterion in acceptance_criteria:
        # 各受入基準からテストケースを生成
        test_case = f"""
    def test_{generate_test_name(criterion)}(self):
        \"\"\"
        受入基準: {criterion}
        \"\"\"
        # Arrange
        {generate_test_arrange(criterion)}

        # Act
        {generate_test_act(criterion)}

        # Assert
        {generate_test_assert(criterion)}
"""
        test_cases.append(test_case)

    return "\n".join(test_cases)


def find_test_insertion_point(test_content):
    """テストの適切な挿入位置を特定"""
    # 最後のテストクラスの最後のメソッドの後ろ
    # または、ファイル末尾
    lines = test_content.split("\n")

    # クラス定義を探す
    for i in range(len(lines) - 1, -1, -1):
        if lines[i].strip().startswith("def test_"):
            # 最後のテストメソッドを見つけた
            # その終わりを探す
            for j in range(i + 1, len(lines)):
                if lines[j].strip() and not lines[j].startswith(" "):
                    # インデントが戻った = メソッド終了
                    return "\n".join(lines[:j])
            # ファイル末尾まで到達
            return test_content

    # テストメソッドが見つからない場合、クラスの最後
    for i in range(len(lines) - 1, -1, -1):
        if lines[i].strip().startswith("class "):
            # クラスの終わりまで進む
            return test_content

    # 何も見つからない場合、ファイル末尾
    return test_content
```

---

#### ステップ1-3: Green フェーズの実装

```python
def execute_green_phase(task, codex_enabled):
    """Green フェーズ: テストを通す最小実装"""

    print(f"  🟢 Green → 実装中...")

    if codex_enabled:
        # Codexモード
        execute_green_phase_with_codex(task)
    else:
        # 通常モード
        execute_green_phase_normal(task)

    # 品質管理ツール実行
    run_quality_checks()

    print(f"  ✅ テスト成功")
    log_step("Green Phase", f"タスク{task['id']}: 実装完了")


def execute_green_phase_normal(task):
    """通常モード: Claude自身が実装を作成"""

    # 実装対象ファイルを特定
    impl_file_path = determine_impl_file_path(task)

    # 実装コードを生成
    impl_code = generate_implementation(task)

    # Bashツール: ls {impl_file_path} 2>/dev/null
    if impl_file_exists:
        # Readツールで既存実装を読み込み
        # Read: impl_file_path

        # 修正対象箇所を特定
        modification_target = find_modification_target(existing_impl_content, task)

        # Editツールで修正
        # Edit: impl_file_path,
        #       old_string=modification_target.old_code,
        #       new_string=impl_code
    else:
        # Writeツールで新規作成
        impl_file_content = generate_impl_file_template(task, impl_code)
        # Write: impl_file_path, impl_file_content

    # テスト実行（成功確認）
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # テスト失敗（想定外）
        # ロールバックを試みる
        rollback_changes(impl_file_path)
        raise TaskSkippableError(f"テスト失敗（Green実装後）: {pytest_output}")


def execute_green_phase_with_codex(task):
    """Codexモード: Codex CLIで実装を生成"""

    # プロンプトファイルを作成
    prompt_content = generate_codex_prompt_for_green(task)

    # Writeツール: /tmp/codex_prompt_{timestamp}.txt
    prompt_file = f"/tmp/codex_prompt_{datetime.now().strftime('%Y%m%d%H%M%S')}.txt"
    # Write: prompt_file, prompt_content

    # Codex CLI実行
    output_file = f"/tmp/codex_output_{datetime.now().strftime('%Y%m%d%H%M%S')}.txt"
    # Bashツール: codex --mode diff --in {prompt_file} > {output_file}

    if codex_exit_code != 0:
        # クリーンアップ
        # Bashツール: rm {prompt_file} {output_file}
        raise TaskSkippableError(f"Codex実行失敗: exit code {codex_exit_code}")

    # 出力読み込み
    # Read: output_file

    # 出力形式の判定
    if is_unified_diff_format(codex_output):
        # unified diff形式 → patchコマンドで適用
        apply_unified_diff(codex_output)
    else:
        # 本文形式 → 直接ファイルに書き込み
        apply_full_content(codex_output, task)

    # フォーマット実行
    # Bashツール（並列実行）:
    # poetry run black src/ tests/
    # poetry run isort src/ tests/

    # テスト実行（成功確認）
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # 失敗時はロールバック
        rollback_changes(affected_files)
        # クリーンアップ
        # Bashツール: rm {prompt_file} {output_file}
        raise TaskSkippableError(f"Codex生成コードでテスト失敗: {pytest_output}")

    # クリーンアップ
    # Bashツール: rm {prompt_file} {output_file}


def apply_unified_diff(diff_content):
    """unified diff形式の差分を適用"""

    # 一時ファイルに差分を保存
    diff_file = f"/tmp/codex_diff_{datetime.now().strftime('%Y%m%d%H%M%S')}.patch"
    # Write: diff_file, diff_content

    # patchコマンドで適用（dry-runで確認）
    # Bashツール: patch -p1 --dry-run < {diff_file}

    if patch_dry_run_exit_code != 0:
        # Bashツール: rm {diff_file}
        raise TaskSkippableError(f"差分適用失敗（dry-run）: patch exit code {patch_dry_run_exit_code}")

    # 本適用
    # Bashツール: patch -p1 < {diff_file}

    if patch_exit_code != 0:
        # Bashツール: rm {diff_file}
        raise TaskSkippableError(f"差分適用失敗: patch exit code {patch_exit_code}")

    # クリーンアップ
    # Bashツール: rm {diff_file}


def apply_full_content(content, task):
    """本文形式の出力を適用"""

    # ファイルパスを特定（コンテンツから抽出またはタスク情報から推定）
    file_path = extract_file_path_from_content(content, task)

    # 既存ファイルかチェック
    # Bashツール: ls {file_path} 2>/dev/null

    if file_exists:
        # Readツールで読み込み（警告チェック）
        # Read: file_path
        # Editツールで全置換
        # Edit: file_path,
        #       old_string=existing_content,
        #       new_string=content
    else:
        # Writeツールで新規作成
        # Write: file_path, content


def determine_impl_file_path(task):
    """タスクから実装ファイルパスを決定"""
    # タスクのメタデータから取得
    impl_path = task.get("impl_file", None)

    if not impl_path:
        # タスクタイトルから推定
        # 例: "UserAuth: ログイン機能実装" → "src/user_auth/login.py"
        impl_path = infer_impl_path_from_task_title(task["title"])

    return impl_path


def generate_implementation(task):
    """タスクから実装コードを生成"""
    # 受入基準と設計情報から実装を生成
    implementation = f"""
def {generate_function_name(task)}({generate_function_params(task)}):
    \"\"\"
    {task['description']}

    Args:
        {generate_function_args_doc(task)}

    Returns:
        {generate_function_return_doc(task)}
    \"\"\"
    {generate_function_body(task)}
"""
    return implementation


def generate_codex_prompt_for_green(task):
    """Greenフェーズ用のCodexプロンプトを生成"""

    # Readツールで関連ファイルを読み込み（状況に応じて）
    # - 設計書（design.md）
    # - 既存の実装ファイル
    # - テストファイル

    prompt = f"""# タスク: {task['title']}

## 説明
{task['description']}

## 受入基準
{format_acceptance_criteria(task['acceptance_criteria'])}

## 実装指示
- テストを通す最小限の実装を作成してください
- 型ヒントを必ず付与してください
- Docstringを記載してください

## 出力形式
unified diff形式（`patch -p1`で適用可能な形式）で出力してください。

---
"""
    return prompt
```

---

#### ステップ1-4: Refactor フェーズの実装

```python
def execute_refactor_phase(task, codex_enabled):
    """Refactor フェーズ: 動作不変の品質改善"""

    print(f"  🔵 Refactor → リファクタリング中...")

    if codex_enabled:
        execute_refactor_phase_with_codex(task)
    else:
        execute_refactor_phase_normal(task)

    print(f"  ✅ テスト成功維持")
    log_step("Refactor Phase", f"タスク{task['id']}: リファクタリング完了")


def execute_refactor_phase_normal(task):
    """通常モード: 安全なリファクタリング"""

    # 実装ファイルを特定
    impl_file_path = determine_impl_file_path(task)

    # Readツールで現在のコードを読み込み
    # Read: impl_file_path

    # リファクタリング箇所を特定
    refactor_suggestions = analyze_code_for_refactoring(current_code)

    if len(refactor_suggestions) == 0:
        print(f"  ℹ️  リファクタリング不要")
        return

    # 最も安全なリファクタリングを適用（1つずつ）
    safe_refactoring = refactor_suggestions[0]

    print(f"  🔧 適用: {safe_refactoring.description}")

    # Editツールで適用
    # Edit: impl_file_path,
    #       old_string=safe_refactoring.old_code,
    #       new_string=safe_refactoring.new_code

    # テスト実行（成功維持確認）
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # 失敗した場合はロールバック
        rollback_changes(impl_file_path)
        log_warning("Step 1", f"リファクタリング失敗（ロールバック済み）: {safe_refactoring.description}")
        # リファクタリング失敗は許容（タスク完了扱い）


def execute_refactor_phase_with_codex(task):
    """Codexモード: Codex CLIでリファクタリング案を生成"""

    # プロンプト作成
    prompt_content = generate_codex_prompt_for_refactor(task)
    prompt_file = f"/tmp/codex_prompt_refactor_{datetime.now().strftime('%Y%m%d%H%M%S')}.txt"
    # Write: prompt_file, prompt_content

    # Codex CLI実行
    output_file = f"/tmp/codex_output_refactor_{datetime.now().strftime('%Y%m%d%H%M%S')}.txt"
    # Bashツール: codex --mode diff --in {prompt_file} > {output_file}

    if codex_exit_code != 0:
        log_warning("Step 1", f"Codex実行失敗（Refactorフェーズ）: exit code {codex_exit_code}")
        # Bashツール: rm {prompt_file} {output_file}
        return  # リファクタリング失敗は許容

    # 出力読み込み
    # Read: output_file

    # 差分適用
    try:
        if is_unified_diff_format(codex_output):
            apply_unified_diff(codex_output)
        else:
            apply_full_content(codex_output, task)
    except Exception as e:
        log_warning("Step 1", f"Codex差分適用失敗: {e}")
        # Bashツール: rm {prompt_file} {output_file}
        return

    # テスト実行
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # 失敗時はロールバック
        rollback_changes(affected_files)
        log_warning("Step 1", f"Codexリファクタリングでテスト失敗（ロールバック済み）")

    # クリーンアップ
    # Bashツール: rm {prompt_file} {output_file}


def analyze_code_for_refactoring(code):
    """コードを分析してリファクタリング候補を抽出"""
    suggestions = []

    # 1. 長い関数の分割
    long_functions = find_long_functions(code)
    for func in long_functions:
        suggestions.append(RefactoringSuggestion(
            description=f"関数{func.name}を分割（{func.lines}行）",
            old_code=func.code,
            new_code=split_function(func),
            safety="medium"
        ))

    # 2. 重複コードの抽出
    duplicates = find_duplicate_code(code)
    for dup in duplicates:
        suggestions.append(RefactoringSuggestion(
            description=f"重複コードを共通化",
            old_code=dup.code,
            new_code=extract_common_function(dup),
            safety="high"
        ))

    # 3. 変数名の改善
    unclear_names = find_unclear_variable_names(code)
    for name_issue in unclear_names:
        suggestions.append(RefactoringSuggestion(
            description=f"変数名を改善: {name_issue.old_name} → {name_issue.new_name}",
            old_code=name_issue.old_code,
            new_code=name_issue.new_code,
            safety="high"
        ))

    # 安全性でソート（high > medium > low）
    suggestions.sort(key=lambda s: {"high": 3, "medium": 2, "low": 1}[s.safety], reverse=True)

    return suggestions


def generate_codex_prompt_for_refactor(task):
    """Refactorフェーズ用のCodexプロンプトを生成"""

    # 実装ファイルを読み込み
    impl_file_path = determine_impl_file_path(task)
    # Read: impl_file_path

    prompt = f"""# タスク: {task['title']} - リファクタリング

## 現在の実装
```python
{current_impl_content}
```

## リファクタリング指示
以下の観点で安全にリファクタリングしてください：
- コードの可読性向上（変数名、関数名の改善）
- 重複コードの削除
- 関数の適切な分割
- 型ヒントの追加・改善

## 制約
- テストの動作を変えないこと（動作不変）
- 既存のインターフェースを維持すること
- 小さな変更に留めること

## 出力形式
unified diff形式（`patch -p1`で適用可能な形式）で出力してください。

---
"""
    return prompt
```

---

#### ステップ1-5: 品質管理ツールの実行

```python
def run_quality_checks():
    """品質管理ツールを実行"""

    # 並列実行（独立したコマンド）
    # Bashツール（4つ並列）:
    # 1. poetry run isort src/ tests/
    # 2. poetry run black src/ tests/
    # 3. poetry run mypy src/
    # 4. poetry run flake8 src/ tests/

    # エラーチェック
    if isort_exit_code != 0:
        log_warning("Quality Check", f"isort failed: exit code {isort_exit_code}")
    if black_exit_code != 0:
        log_warning("Quality Check", f"black failed: exit code {black_exit_code}")
    if mypy_exit_code != 0:
        log_warning("Quality Check", f"mypy failed: exit code {mypy_exit_code}")
    if flake8_exit_code != 0:
        log_warning("Quality Check", f"flake8 failed: exit code {flake8_exit_code}")
```

---

#### ステップ1-6: 状態更新ヘルパー関数

```python
def update_state_phase(state_file_path, new_phase):
    """state.jsonのphaseフィールドを更新"""

    # Readツールで現在の状態を読み込み
    # Read: state_file_path

    old_state = current_state_content
    state = json.loads(old_state)
    state["phase"] = new_phase
    state["updated_at"] = datetime.now().isoformat()

    # Editツールで更新
    # Edit: state_file_path,
    #       old_string=old_state,
    #       new_string=json.dumps(state, indent=2, ensure_ascii=False)


def complete_current_task(state_file_path, task_id):
    """現在のタスクを完了扱いにする"""

    # Readツールで状態読み込み
    # Read: state_file_path

    old_state = current_state_content
    state = json.loads(old_state)

    # pending から completed に移動
    state["tasks"]["pending"].remove(task_id)
    state["tasks"]["completed"].append(task_id)

    # 次のタスクに進む
    if len(state["tasks"]["pending"]) > 0:
        state["current_task"] = state["tasks"]["pending"][0]
        state["phase"] = "red"

    state["updated_at"] = datetime.now().isoformat()

    # Editツールで更新
    # Edit: state_file_path,
    #       old_string=old_state,
    #       new_string=json.dumps(state, indent=2, ensure_ascii=False)


def skip_current_task(state_file_path, task_id):
    """現在のタスクをスキップ扱いで完了"""

    # complete_current_task と同じ処理
    # （完了扱いだが、警告ログに記録済み）
    complete_current_task(state_file_path, task_id)


def parse_task_from_markdown(tasks_md_content, task_id):
    """tasks.mdから指定IDのタスクを抽出"""

    # Markdown形式のタスクリストをパース
    # 例:
    # ## タスク 0: ユーザー認証: インターフェース定義
    # **説明**: ...
    # **受入基準**:
    # - ...

    lines = tasks_md_content.split("\n")
    current_task = None
    in_task = False

    for line in lines:
        if line.startswith(f"## タスク {task_id}:"):
            in_task = True
            current_task = {
                "id": task_id,
                "title": line.replace(f"## タスク {task_id}:", "").strip(),
                "description": "",
                "acceptance_criteria": []
            }
        elif in_task:
            if line.startswith("**説明**:"):
                current_task["description"] = line.replace("**説明**:", "").strip()
            elif line.startswith("- "):
                current_task["acceptance_criteria"].append(line[2:].strip())
            elif line.startswith("## タスク"):
                # 次のタスクに到達
                break

    return current_task
```

---

#### ステップ1-7: ロールバック処理

```python
def rollback_changes(file_paths):
    """変更をロールバック（git checkoutを使用）"""

    if isinstance(file_paths, str):
        file_paths = [file_paths]

    for file_path in file_paths:
        # Bashツール: git checkout HEAD -- {file_path}
        print(f"  🔄 ロールバック: {file_path}")
        log_step("Rollback", f"ロールバック実行: {file_path}")
```

---

#### ステップ1-8: メインループ完了時のログ記録

```python
# メインループ終了後
end_time = datetime.now()
execution_duration = end_time - start_time

log_step("Step 1", f"""全タスク自動実装完了
実行ステップ数: {step_count}
完了タスク数: {len(completed_tasks)}
スキップタスク数: {len(skipped_tasks)}
実行時間: {execution_duration}
スキップ詳細:
{format_skipped_tasks(skipped_tasks)}
""")
```

**進捗報告**:
```
✅ Step 1: 全タスク自動実装完了
   実行ステップ数: {step_count}
   完了タスク数: {len(completed_tasks)}
   スキップタスク数: {len(skipped_tasks)}
   実行時間: {execution_duration}
```

---

### Step 2: 最終品質チェック

全タスク完了後の品質検証を実行します。

#### ステップ2-1: 全テスト実行

```bash
# Bashツール（順次実行）:
# 1. poetry run pytest -q
# 2. poetry run pytest --cov=src --cov-report=term-missing
```

```python
print("\n🔍 Step 2: 最終品質チェック")

# テスト実行結果の確認
if pytest_exit_code != 0:
    log_warning("Step 2", f"テスト失敗:\n{pytest_output}")
    print(f"⚠️  テスト失敗（詳細はログ参照）")
else:
    print(f"✅ 全テスト成功")

# カバレッジ抽出
coverage_percentage = extract_coverage_from_output(pytest_cov_output)
print(f"✅ カバレッジ: {coverage_percentage}%")
```

---

#### ステップ2-2: 品質管理ツール実行

```bash
# Bashツール（並列実行）:
# 1. poetry run isort src/ tests/
# 2. poetry run black src/ tests/
# 3. poetry run mypy src/
# 4. poetry run flake8 src/ tests/
```

```python
# 各ツールの結果確認
quality_check_results = {
    "isort": isort_exit_code == 0,
    "black": black_exit_code == 0,
    "mypy": mypy_exit_code == 0,
    "flake8": flake8_exit_code == 0
}

for tool, success in quality_check_results.items():
    if success:
        print(f"✅ {tool}完了")
    else:
        log_warning("Step 2", f"{tool}エラー:\n{tool_output}")
        print(f"⚠️  {tool}エラー（詳細はログ参照）")
```

**ログ記録**:
```python
log_step("Step 2", f"""最終品質チェック完了
テスト: {"✅ 全パス" if pytest_exit_code == 0 else "⚠️ 失敗"}
カバレッジ: {coverage_percentage}%
フォーマット: {"✅" if quality_check_results["black"] else "⚠️"}
型チェック: {"✅" if quality_check_results["mypy"] else "⚠️"}
リント: {"✅" if quality_check_results["flake8"] else "⚠️"}
""")
```

**進捗報告**:
```
✅ Step 2: 最終品質チェック完了
   テスト: {"✅ 全パス" if pytest_exit_code == 0 else "⚠️ 失敗"}
   カバレッジ: {coverage_percentage}%
   フォーマット: {"✅" if quality_check_results["black"] else "⚠️"}
   型チェック: {"✅" if quality_check_results["mypy"] else "⚠️"}
   リント: {"✅" if quality_check_results["flake8"] else "⚠️"}
```

---

### Step 3: 自動コミット

機能単位で自動コミットを実行します。

#### ステップ3-1: 変更検出

```bash
# Bashツール（並列実行）:
# 1. git status --porcelain
# 2. git diff --stat
```

```python
print("\n📝 Step 3: 自動コミット")

changed_files = parse_git_status(git_status_output)
diff_summary = git_diff_output

if len(changed_files) == 0:
    print("ℹ️  変更なし。コミットをスキップします。")
    log_step("Step 3", "変更なし（コミットスキップ）")
    # Step 4へスキップ
else:
    print(f"変更ファイル数: {len(changed_files)}個")
```

---

#### ステップ3-2: コミットメッセージ生成

```python
# 仕様書モードの場合、spec.jsonとdesign.mdから情報抽出
if SPEC_MODE:
    # すでにStep 0で読み込み済みなら再利用
    # そうでなければReadツールで読み込み
    # Read: SOURCE_SPEC
    # Read: f"{SPEC_DIR}/design.md"

    # Conventional Commits形式でメッセージ生成
    commit_type = determine_commit_type(spec_json, design_content)
    # feat, fix, refactor, test, docs, etc.

    scope = FEATURE_NAME
    subject = spec_json['description']

    # 変更の詳細を抽出
    body_lines = []
    for section in ["Components & Interfaces", "Data Models", "System Flows"]:
        if section in design_content:
            summary = summarize_section(design_content, section)
            body_lines.extend(summary)

    body = "\n".join(body_lines)
else:
    # 仕様書なしモード: タスクリストから推定
    commit_type = "feat"
    scope = FEATURE_NAME
    subject = f"{FEATURE_NAME}の実装"
    body = generate_commit_body_from_tasks(tasks_md_content)

footer = """
🤖 Generated with Claude Code (https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
"""

commit_message = f"""{commit_type}({scope}): {subject}

{body}

{footer}"""

print(f"コミットメッセージ: {commit_message.split(chr(10))[0]}")
```

---

#### ステップ3-3: Gitコミット実行

```bash
# Bashツール（順次実行）:
# 1. git add .
# 2. git commit -m "$(cat <<'EOF'
#    {commit_message}
#    EOF
#    )"
```

```python
if git_commit_exit_code != 0:
    log_error("Step 3", f"gitコミット失敗:\n{git_commit_output}")

# コミットハッシュ取得
# Bashツール: git rev-parse HEAD
commit_hash = git_rev_parse_output.strip()

print(f"✅ コミット成功: {commit_hash[:7]}")
```

---

#### ステップ3-4: コミット情報の保存（オプション）

```python
# .claude/commits/ディレクトリに保存
commit_info_file = f".claude/commits/{commit_hash[:7]}_{FEATURE_NAME}.md"

commit_info_content = f"""# Commit: {commit_hash[:7]}

## メッセージ
```
{commit_message}
```

## 変更ファイル
{git_diff_output}

## 実装情報
- 機能: {FEATURE_NAME}
- タスク数: {total_tasks}
- 完了タスク: {len(completed_tasks)}
- スキップタスク: {len(skipped_tasks)}
- 実行ステップ数: {step_count}

## 仕様書
{SOURCE_SPEC if SPEC_MODE else "なし"}
{f"{SPEC_DIR}/design.md" if SPEC_MODE else ""}
"""

# Bashツール: mkdir -p .claude/commits
# Writeツール: commit_info_file, commit_info_content
```

**ログ記録**:
```python
log_step("Step 3", f"""自動コミット完了
Message: {commit_message.split(chr(10))[0]}
Commit Hash: {commit_hash}
Files Changed: {len(changed_files)}個
""")
```

**進捗報告**:
```
✅ Step 3: 自動コミット完了
   Message: {commit_message.split(chr(10))[0]}
   Commit Hash: {commit_hash}
   Files Changed: {len(changed_files)}個
```

---

### Step 4: 最終報告

```python
print("\n" + "="*60)
print("✅ flow-complete が正常に完了しました")
print("="*60)

# 終了時刻を記録
end_time = datetime.now()
total_execution_duration = end_time - start_time

# 最終報告の生成
final_report = f"""
【実行サマリ】
機能: {FEATURE_NAME}
実行時間: {start_time.strftime('%H:%M:%S')} → {end_time.strftime('%H:%M:%S')} ({total_execution_duration})
実行ステップ数: {step_count}

【タスク実行結果】
全タスク: {total_tasks}個
完了タスク: {len(completed_tasks)}個
スキップタスク: {len(skipped_tasks)}個
完了率: {(len(completed_tasks) / total_tasks * 100):.1f}%

【品質チェック結果】
- テスト: {"✅ 全パス" if pytest_exit_code == 0 else "⚠️ 失敗"}
- カバレッジ: {coverage_percentage}%
- フォーマット: {"✅ 完了" if quality_check_results["black"] else "⚠️ エラー"}
- 型チェック: {"✅ 完了" if quality_check_results["mypy"] else "⚠️ エラー"}
- リント: {"✅ 完了" if quality_check_results["flake8"] else "⚠️ エラー"}

{"【コミット結果】" if len(changed_files) > 0 else ""}
{f"Message: {commit_message.split(chr(10))[0]}" if len(changed_files) > 0 else ""}
{f"Commit Hash: {commit_hash}" if len(changed_files) > 0 else ""}
{f"Files Changed: {len(changed_files)}個" if len(changed_files) > 0 else ""}

{"【警告ログ】" if len(skipped_tasks) > 0 else ""}
{format_skipped_tasks(skipped_tasks) if len(skipped_tasks) > 0 else ""}

【詳細ログ】
{LOG_FILE}
"""

print(final_report)

# ログに記録
log_step("Step 4", f"""最終報告完了
実行時間: {total_execution_duration}
完了タスク数: {len(completed_tasks)}
スキップタスク数: {len(skipped_tasks)}
{"コミット: " + commit_hash if len(changed_files) > 0 else "コミットなし"}
""")


def format_skipped_tasks(skipped_tasks):
    """スキップタスクを整形して表示"""
    if len(skipped_tasks) == 0:
        return ""

    lines = []
    for task in skipped_tasks:
        lines.append(f"- タスク{task['task_id']} 「{task['title']}」: {task['reason']}")

    return "\n".join(lines)
```

---

## エラーハンドリング

### エラー分類と対応

#### 1. 前提条件エラー（即時終了）

```python
class PreconditionError(Exception):
    """前提条件を満たさないエラー"""
    pass

# 対応:
# - state.jsonが存在しない → log_error() → exit(1)
# - tasks.mdが存在しない → log_error() → exit(1)
```

#### 2. ファイル操作エラー（即時終了）

```python
class FileOperationError(Exception):
    """ファイル読み書きエラー"""
    pass

# 対応:
# - Read/Write/Edit失敗 → log_error() → exit(1)
```

#### 3. タスク実行エラー（スキップ可能）

```python
class TaskSkippableError(Exception):
    """スキップ可能なタスクエラー"""
    pass

# 対応:
# - テスト恒久失敗 → log_warning() → skip_current_task() → 継続
# - 差分規模超過 → log_warning() → skip_current_task() → 継続
# - Codex実行失敗 → log_warning() → skip_current_task() → 継続
```

#### 4. 状態不整合エラー（即時終了）

```python
class StateInconsistencyError(Exception):
    """状態不整合エラー"""
    pass

# 対応:
# - Redフェーズでテスト成功 → log_error() → exit(1)
# - state.jsonが破損 → log_error() → exit(1)
```

#### 5. Git操作エラー（即時終了）

```python
class GitOperationError(Exception):
    """Git操作エラー"""
    pass

# 対応:
# - git commit失敗 → log_error() → exit(1)
```

---

## 安全装置

### 1. 最大反復回数

```python
MAX_STEPS = 100
# Step 1で100ステップ到達時は警告を出して終了
```

### 2. 変更規模上限

```python
MAX_FILES_NORMAL = 3
MAX_LINES_NORMAL = 120
MAX_LINES_CODEX = 200

# 超過時はTaskSkippableErrorをraise
```

### 3. ロールバック機能

```python
# テスト失敗時は自動的にgit checkoutでロールバック
def rollback_changes(file_paths):
    for file_path in file_paths:
        # Bashツール: git checkout HEAD -- {file_path}
        pass
```

---

## 使用例

### 基本的な使い方

```bash
# 前提: /flow-init で初期化済み
/flow-complete
```

### Codex連携

```bash
# 前提: /flow-init で初期化済み
/flow-complete --codex
```

---

## 制限事項

- `/flow-init`でTDDフローが初期化済みであることが必須
- 最大反復回数: 100ステップ
- タスク単位でのエラーは警告ログに記録し、継続実行
- 状態不整合エラーは全体停止
- コミット失敗時は全体停止

---

## 想定実行時間

- **小規模**（残タスク: 3-5個）: 10-20分
- **中規模**（残タスク: 10-15個）: 30-60分
- **大規模**（残タスク: 20個以上）: 1-2時間

※Codexモード使用時は、API呼び出しのオーバーヘッドにより、通常の1.5-2倍の時間がかかる可能性があります。

---

## 実装チェックリスト

### Step 0
- [ ] 2つのコマンド定義ファイルを並列読み込み（flow-next.md, ccommit.md）
- [ ] 引数解析（--codex検出）
- [ ] 前提条件チェック（state.json, tasks.md並列読み込み）
- [ ] 仕様書モードの確認（条件付き並列読み込み）
- [ ] ログファイル初期化（Writeツール）
- [ ] 推定実行時間の算出

### Step 1
- [ ] メインループ（while True）
- [ ] state.json読み込み（各イテレーション）
- [ ] tasks.md読み込み（各イテレーション）
- [ ] Red/Green/Refactorフェーズ実行
- [ ] 品質管理ツール実行（並列）
- [ ] 状態更新（Editツール）
- [ ] エラーハンドリング（スキップ/ロールバック）
- [ ] ログ記録

### Step 2
- [ ] 全テスト実行（Bashツール順次）
- [ ] カバレッジ取得
- [ ] 品質管理ツール実行（並列）
- [ ] ログ記録

### Step 3
- [ ] 変更検出（Bashツール並列）
- [ ] 仕様書モードの分岐（spec.json/design.md読み込み）
- [ ] コミットメッセージ生成
- [ ] gitコミット実行（Bashツール順次）
- [ ] コミット情報保存（Writeツール）
- [ ] ログ記録

### Step 4
- [ ] 最終報告生成
- [ ] ログ記録

---

## バージョン履歴

- **v2.0**: 超詳細版（ツール使用の具体化、並列実行指示、状態管理詳細化、Codex連携詳細化、エラーハンドリング強化）
- **v1.0**: 初版
