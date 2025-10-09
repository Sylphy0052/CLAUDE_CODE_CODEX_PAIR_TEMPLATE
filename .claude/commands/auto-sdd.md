---
description: "[完全自動SDD実行] Steeringから実装・コミットまで全自動実行。機能説明または仕様書パスを引数に指定。"
argument_hint: "<feature-description-or-spec-path> [--codex (任意)]"
notes: |
  バージョン: 3.0（完全自動実行・放置前提設計・超詳細版）
  依存: steering, spec-init, spec-requirements, spec-design, validate-design, flow-init, flow-next, ccommit
  前提: プロジェクトがGitリポジトリであること
  実行時間: 小規模10-20分、中規模30-60分、大規模1-2時間
---
# 完全自動SDD実行コマンド (auto-sdd) - 超詳細版

## 概要

このコマンドは、**仕様駆動開発(SDD)の全工程を完全自動で実行**します。
Steeringの更新から要件定義、設計、検証、実装、コミットまで、ユーザーの承認なしで一気通貫で処理します。

**設計思想**:

- **完全自動実行**: ユーザーの承認を一切求めない（放置前提）
- **段階的詳細化**: 要件→設計→タスク→実装の順で段階的に進める
- **品質重視**: 各フェーズで品質チェックを実施し、エラーは自動修正または警告
- **エラー耐性**: 個別タスクのエラーで全体を停止させない
- **透明性**: 各フェーズの進捗を詳細に報告
- **ツール直接使用**: Read/Write/Edit/Bashツールを直接実行（サブエージェント不使用）

**引数**:

- `$1`: 機能の自然言語説明（例: "ユーザー認証機能"）または既存の仕様書パス（`.claude/specs/[feature]/spec.json`）
- `--codex`: Codex連携を有効化（全実装ステップで適用）

**実行前の注意**:
- このコマンドは長時間実行される可能性があります（10分〜2時間）
- 実行中は他の操作を行わないでください
- 未コミットの変更がある場合は事前にコミットまたはstashしてください
- Gitリポジトリであることが必須です

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

### Phase 0: 事前チェックと初期化

このフェーズでは、実行環境の確認と引数の解析を行います。

#### ステップ0-1: コマンド定義ファイルの読み込み（並列実行）

**必須操作**: Readツールを並列で使って以下のファイルを確認

```
複数のReadツールを1つのメッセージで実行:
- .claude/commands/steering.md
- .claude/commands/spec-init.md
- .claude/commands/spec-requirements.md
- .claude/commands/spec-design.md
- .claude/commands/validate-design.md
- .claude/commands/flow-init.md
- .claude/commands/flow-next.md
- .claude/commands/ccommit.md
```

**目的**: 各コマンドの最新仕様を理解し、正確に実行する

**進捗報告**:
```
📖 Phase 0-1: コマンド定義ファイルを読み込み中...
✅ 8個のコマンド定義を読み込み完了
```

---

#### ステップ0-2: 引数の解析と変数初期化

**処理手順**:

1. **引数の取得**
   ```python
   # 疑似コードで論理を示す
   ARGS = "$*"  # 全引数を取得
   CODEX_FLAG = ""
   FEATURE_INPUT = ""
   MODE = ""
   ```

2. **--codexオプションの検出**
   ```python
   if "--codex" in ARGS:
       CODEX_FLAG = "--codex"
       CODEX_ENABLED = True
       FEATURE_INPUT = ARGS.replace("--codex", "").strip()
   else:
       CODEX_ENABLED = False
       FEATURE_INPUT = ARGS.strip()
   ```

3. **引数タイプの判定**
   ```python
   # パス形式の判定
   if (".claude/specs/" in FEATURE_INPUT or
       FEATURE_INPUT.endswith("/spec.json") or
       os.path.exists(FEATURE_INPUT)):
       MODE = "existing-spec"
       SPEC_PATH = FEATURE_INPUT
       # spec.jsonを読み込んで機能名を取得
       # Readツール実行: SPEC_PATH
   else:
       MODE = "new-feature"
       FEATURE_DESCRIPTION = FEATURE_INPUT
   ```

4. **引数の検証**
   ```python
   if not FEATURE_INPUT:
       # エラー終了
       print("❌ エラー: 引数が空です")
       print("使用例: /auto-sdd \"機能説明\" または /auto-sdd .claude/specs/[feature]/spec.json")
       exit(1)

   if MODE == "existing-spec" and not os.path.exists(SPEC_PATH):
       # エラー終了
       print(f"❌ エラー: 指定された仕様書が存在しません: {SPEC_PATH}")
       exit(1)

   if MODE == "new-feature" and len(FEATURE_DESCRIPTION) < 5:
       # 警告表示
       print(f"⚠️  警告: 機能説明が短すぎます（{len(FEATURE_DESCRIPTION)}文字）")
       print("💡 推奨: より詳細な説明を提供してください")
   ```

**進捗報告**:
```
📋 Phase 0-2: 引数解析完了
   モード: [新規機能開発 | 既存仕様書からの実装]
   入力: [FEATURE_INPUT]
   Codex連携: [✅ 有効 | ❌ 無効]
```

---

#### ステップ0-3: 前提条件の確認（並列実行）

**Bashツールを並列で実行**（独立したチェック）:

1. **Gitリポジトリチェック**
2. **未コミット変更の確認**
3. **Poetry環境の確認**（pyproject.toml存在時のみ）

```bash
# 並列実行1: Gitリポジトリチェック
git rev-parse --is-inside-work-tree

# 並列実行2: 未コミット変更の確認
git status --porcelain

# 並列実行3: Poetry環境確認（条件付き）
if [[ -f "pyproject.toml" ]]; then
  poetry env info
fi
```

**エラーハンドリング**:

```python
# Git check
if git_check_exit_code != 0:
    print("❌ エラー: このディレクトリはGitリポジトリではありません")
    print("💡 解決方法: git init を実行してください")
    exit(1)

# Uncommitted changes
uncommitted = git_status_output.strip()
if uncommitted:
    print("⚠️  警告: 未コミットの変更があります")
    print(uncommitted)
    print("")
    print("💡 推奨: 変更をコミットまたはstashしてから実行してください")
    print("   - git add . && git commit -m \"作業中の変更を保存\"")
    print("   - git stash")
    print("")
    print("⏳ 5秒後に自動で続行します（Ctrl+Cで中断）...")
    # Bashツールでsleep 5を実行

# Poetry check
if pyproject.toml exists and poetry_check_exit_code != 0:
    print("⚠️  警告: Poetry環境が初期化されていません")
    print("💡 自動で初期化します...")
    # Bashツールでpoetry installを実行
```

**ディレクトリ構造の確認**:
```bash
# Bashツール実行（順次）
mkdir -p src tests .claude
```

**進捗報告**:
```
✅ Phase 0-3: 前提条件チェック完了
   - Gitリポジトリ: ✅
   - 未コミット変更: [あり | なし]
   - Poetry環境: ✅
   - ディレクトリ構造: ✅
```

---

#### ステップ0-4: 実行ログの初期化

**Writeツールで新規ログファイルを作成**:

```python
from datetime import datetime

LOG_DIR = ".claude/.logs"
TIMESTAMP = datetime.now().strftime("%Y%m%d_%H%M%S")
LOG_FILE = f"{LOG_DIR}/auto-sdd_{TIMESTAMP}.log"

# Bashツールでディレクトリ作成
# mkdir -p .claude/.logs

# Writeツールでログファイル作成
log_content = f"""# 完全自動SDD実行ログ
実行開始: {datetime.now().isoformat()}
モード: {MODE}
入力: {FEATURE_INPUT}
Codex連携: {CODEX_ENABLED}

────────────────────────────────────────────────────────
"""
# Write: LOG_FILE, log_content
```

**ヘルパー関数の定義**（メモリ内）:

```python
# 以下はメモリ内で保持（実際にファイルは作成しない）

def log_phase(phase_name, message):
    """フェーズ情報をログに追記"""
    log_entry = f"""
[{datetime.now().isoformat()}] {phase_name}
{message}
"""
    # Editツールでログファイルに追記
    # または、ログ全体を再構築してWriteツールで上書き

def log_error(phase, error_msg):
    """エラー情報をログに記録して終了"""
    error_entry = f"""
❌ エラー発生
フェーズ: {phase}
エラー: {error_msg}
終了時刻: {datetime.now().isoformat()}
"""
    # Editツールでログファイルに追記
    print(f"\n❌ エラーが発生しました")
    print(f"フェーズ: {phase}")
    print(f"エラー: {error_msg}")
    print(f"\n📝 詳細ログ: {LOG_FILE}")
    exit(1)

def log_warning(phase, warning_msg):
    """警告情報をログに記録（継続）"""
    warning_entry = f"""
⚠️  警告
フェーズ: {phase}
警告: {warning_msg}
"""
    # Editツールでログファイルに追記
    print(f"\n⚠️  警告")
    print(f"フェーズ: {phase}")
    print(f"警告: {warning_msg}\n")
```

**進捗報告**:
```
📝 Phase 0-4: 実行ログ初期化完了
   ログファイル: {LOG_FILE}
```

---

#### ステップ0-5: 実行開始の宣言

**進捗報告**:
```
╔════════════════════════════════════════════════════════════╗
║  🚀 完全自動SDD実行を開始します                            ║
╚════════════════════════════════════════════════════════════╝

【実行モード】
  モード: [新規機能開発 | 既存仕様書からの実装]
  入力: {FEATURE_INPUT}
  Codex連携: [✅ 有効 | ❌ 無効]

【実行フェーズ】
  Phase 1: プロジェクト知識管理（Steering更新）
  Phase 2: 仕様初期化 [新規機能モードのみ]
  Phase 3: 要件定義生成（EARS形式）
  Phase 4: 設計書生成（包括的設計）
  Phase 5: 設計書品質レビュー（GO/NO-GO判定）
  Phase 6: TDDフロー初期化（タスク自動生成）
  Phase 7: 全タスク自動実装（Red→Green→Refactorループ）
  Phase 8: 最終品質チェック（pytest/mypy/flake8）
  Phase 9: 自動コミット（Conventional Commits）
  Phase 10: 最終報告

【推定実行時間】
  タスク数未確定のため、Phase 6で算出します

【開始時刻】
  {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}

════════════════════════════════════════════════════════════
```

---

### Phase 1: プロジェクト知識管理（Steering更新）

#### 実行内容（steering.mdに準拠）

**ステップ1-1: 既存Steeringファイルの確認（並列読み込み）**

```python
# Readツールを並列で実行
files_to_read = []
steering_dir = ".claude/steering"

# Bashツールでディレクトリ確認
# ls .claude/steering/ 2>/dev/null || echo "not_exists"

# 並列でReadツール実行
if steering_dir_exists:
    files_to_read = [
        f"{steering_dir}/product.md",
        f"{steering_dir}/tech.md",
        f"{steering_dir}/structure.md"
    ]
    # Read: 上記3ファイルを並列で読み込み
```

**ステップ1-2: プロジェクトコンテキストの収集（並列読み込み）**

```python
# Bashツールでディレクトリ構造取得
# ls -la

# 主要ファイルを並列で読み込み
context_files = []
# Bashツールで存在確認
# ls README.md pyproject.toml package.json 2>/dev/null

# 存在するファイルをReadツールで並列読み込み
if "README.md" exists:
    context_files.append("README.md")
if "pyproject.toml" exists:
    context_files.append("pyproject.toml")
if "package.json" exists:
    context_files.append("package.json")

# Read: context_files を並列読み込み
```

**ステップ1-3: Steeringファイルの生成・更新**

```python
# 各Steeringファイルの内容を分析して生成
# - product.md: プロダクト概要、主要機能、ターゲット
# - tech.md: 技術スタック、アーキテクチャ、開発環境
# - structure.md: ディレクトリ構造、命名規則、設計パターン

for file_name in ["product.md", "tech.md", "structure.md"]:
    file_path = f"{steering_dir}/{file_name}"

    if file already exists (from Step 1-1):
        # 既存ファイルを保護しながら更新
        # Editツールで事実情報のみ更新
        # ユーザーカスタマイズは保持
        # 削除項目は[DEPRECATED]でマーク
        # Edit: file_path, old_string, new_string
    else:
        # 新規作成
        # Writeツールでファイル作成
        # Write: file_path, content
```

**ログ記録**:
```python
log_phase("Phase 1", """Steering更新完了
- product.md: [新規作成 | 更新]
- tech.md: [新規作成 | 更新]
- structure.md: [新規作成 | 更新]
""")
```

**進捗報告**:
```
📚 Phase 1: プロジェクト知識管理
✅ Steering更新完了
   - product.md: [新規作成 | 更新]
   - tech.md: [新規作成 | 更新]
   - structure.md: [新規作成 | 更新]
```

---

### Phase 2: 仕様初期化（新規機能モードのみ）

**条件分岐**:
```python
if MODE == "existing-spec":
    print("📦 Phase 2: スキップ（既存仕様書モード）")
    # 既存のspec.jsonをReadして機能名を取得
    # Read: SPEC_PATH
    FEATURE_NAME = spec_json["feature_name"]
    SPEC_DIR = os.path.dirname(SPEC_PATH)
else:
    # 新規機能モードの処理を実行
```

#### 実行内容（spec-init.mdに準拠）

**ステップ2-1: 機能名の生成**

```python
# 自然言語説明からケバブケース形式の機能名を生成
# 例: "ユーザー認証機能" → "user-authentication"

FEATURE_DESCRIPTION = FEATURE_INPUT
FEATURE_NAME = generate_kebab_case(FEATURE_DESCRIPTION)
# 例: スペースを"-"に、日本語をローマ字に、記号を除去
```

**ステップ2-2: 重複チェック**

```python
# Bashツールでチェック
# ls .claude/specs/{FEATURE_NAME} 2>/dev/null

if directory_exists:
    log_error("Phase 2", f"機能名が重複しています: {FEATURE_NAME}")
```

**ステップ2-3: ディレクトリ作成とspec.json生成**

```python
SPEC_DIR = f".claude/specs/{FEATURE_NAME}"

# Bashツールでディレクトリ作成
# mkdir -p {SPEC_DIR}

# spec.jsonの内容を生成
spec_json_content = {
    "feature_name": FEATURE_NAME,
    "description": FEATURE_DESCRIPTION,
    "created_at": datetime.now().isoformat(),
    "updated_at": datetime.now().isoformat(),
    "language": "ja",
    "phase": "initialized",
    "has_requirements": False,
    "has_design": False
}

# Writeツールでspec.json作成
# Write: f"{SPEC_DIR}/spec.json", json.dumps(spec_json_content, indent=2, ensure_ascii=False)
```

**ログ記録**:
```python
log_phase("Phase 2", f"""仕様初期化完了
機能名: {FEATURE_NAME}
パス: {SPEC_DIR}/spec.json
""")
```

**進捗報告**:
```
📦 Phase 2: 仕様初期化
✅ 仕様を初期化
   機能名: {FEATURE_NAME}
   パス: {SPEC_DIR}/spec.json
```

---

### Phase 3: 要件定義生成

#### 実行内容（spec-requirements.mdに準拠）

**ステップ3-1: コンテキストの読み込み（並列実行）**

```python
# 並列でReadツール実行
context_files = [
    f"{SPEC_DIR}/spec.json",
    ".claude/steering/product.md",
    ".claude/steering/tech.md",
    ".claude/steering/structure.md"
]
# Read: context_files を並列読み込み
```

**ステップ3-2: EARS形式要件定義書の生成**

```python
# コンテキストから要件を抽出・構造化
# - 機能の目的と背景
# - 機能要件（EARS形式）
# - 非機能要件
# - 制約条件
# - 受入基準（EARS形式）

# EARS形式パターン:
# - WHEN [イベント] THEN [システム] SHALL [応答]
# - IF [条件] THEN [システム] SHALL [応答]
# - WHILE [継続条件] THE [システム] SHALL [動作]
# - WHERE [コンテキスト] THE [システム] SHALL [動作]

requirements_md_content = generate_requirements_document(
    spec_json,
    steering_product,
    steering_tech,
    steering_structure
)

# Writeツールでrequirements.md作成
# Write: f"{SPEC_DIR}/requirements.md", requirements_md_content
```

**ステップ3-3: spec.jsonの更新**

```python
# Readツールでspec.jsonを再読込（最新状態取得）
# Read: f"{SPEC_DIR}/spec.json"

# 更新内容を準備
old_spec_json = current_spec_json_content
updated_spec_json = json.loads(old_spec_json)
updated_spec_json["phase"] = "requirements"
updated_spec_json["has_requirements"] = True
updated_spec_json["updated_at"] = datetime.now().isoformat()

# Editツールで更新
# Edit: f"{SPEC_DIR}/spec.json",
#       old_string=old_spec_json,
#       new_string=json.dumps(updated_spec_json, indent=2, ensure_ascii=False)
```

**ログ記録**:
```python
log_phase("Phase 3", f"""要件定義生成完了
パス: {SPEC_DIR}/requirements.md
要件数: {requirements_count}個
受入基準: {acceptance_criteria_count}項目（EARS形式）
""")
```

**進捗報告**:
```
📋 Phase 3: 要件定義生成
✅ 要件定義書を生成
   パス: {SPEC_DIR}/requirements.md
   要件数: {requirements_count}個
   受入基準: {acceptance_criteria_count}項目（EARS形式）
```

---

### Phase 4: 設計書生成

#### 実行内容（spec-design.mdに準拠）

**ステップ4-1: コンテキストの読み込み（並列実行）**

```python
# 並列でReadツール実行
context_files = [
    f"{SPEC_DIR}/spec.json",
    f"{SPEC_DIR}/requirements.md",
    ".claude/steering/product.md",
    ".claude/steering/tech.md",
    ".claude/steering/structure.md"
]
# Read: context_files を並列読み込み
```

**ステップ4-2: Discovery & Analysis**

```python
# 1. 機能分類の判定
feature_type = classify_feature(requirements, steering)
# - Greenfield: 完全新規機能
# - Extension: 既存機能の拡張
# - Simple: シンプルな機能
# - Complex: 複雑な機能

# 2. 既存実装の分析（Extension の場合）
if feature_type == "Extension":
    # Grepツールで既存コードを検索
    # Grep: pattern=<関連キーワード>, output_mode="files_with_matches"
    # 見つかったファイルをReadツールで読み込み
    pass

# 3. Steeringとの整合性チェック
check_steering_consistency(requirements, steering)

# 4. 技術調査（必要に応じて）
if needs_external_research:
    # WebSearchまたはWebFetchツールで調査
    # WebSearch: query="<技術調査クエリ>"
    pass

# 5. リスク評価
risks = evaluate_risks(requirements, feature_type, existing_code)
```

**ステップ4-3: 包括的設計書の生成**

```python
# 設計書の各セクションを生成
design_sections = {
    "Overview": generate_overview(),
    "Architecture": generate_architecture_with_mermaid(),
    "Technology Stack & Design Decisions": generate_tech_stack(),
    "System Flows": generate_system_flows_with_mermaid(),
    "Components & Interfaces": generate_components(),
    "Data Models": generate_data_models(),
    "Error Handling": generate_error_handling(),
    "Testing Strategy": generate_testing_strategy(),
}

# 条件付きセクション
if needs_security_section:
    design_sections["Security Considerations"] = generate_security()
if needs_performance_section:
    design_sections["Performance & Scalability"] = generate_performance()
if feature_type == "Extension":
    design_sections["Migration Strategy"] = generate_migration()

design_md_content = format_design_document(design_sections)

# Writeツールでdesign.md作成
# Write: f"{SPEC_DIR}/design.md", design_md_content
```

**ステップ4-4: spec.jsonの更新**

```python
# Readツールでspec.jsonを再読込
# Read: f"{SPEC_DIR}/spec.json"

# 更新
old_spec_json = current_spec_json_content
updated_spec_json = json.loads(old_spec_json)
updated_spec_json["phase"] = "design"
updated_spec_json["has_design"] = True
updated_spec_json["updated_at"] = datetime.now().isoformat()

# Editツールで更新
# Edit: f"{SPEC_DIR}/spec.json",
#       old_string=old_spec_json,
#       new_string=json.dumps(updated_spec_json, indent=2, ensure_ascii=False)
```

**ログ記録**:
```python
log_phase("Phase 4", f"""設計書生成完了
パス: {SPEC_DIR}/design.md
セクション: {len(design_sections)}個
Mermaid図: {mermaid_diagram_count}個
""")
```

**進捗報告**:
```
🏗️ Phase 4: 設計書生成
✅ 包括的な技術設計書を生成
   パス: {SPEC_DIR}/design.md
   セクション: {len(design_sections)}個
   Mermaid図: {mermaid_diagram_count}個
```

---

### Phase 5: 設計書品質レビュー

#### 実行内容（validate-design.mdに準拠）

**ステップ5-1: コンテキストの読み込み（並列実行）**

```python
# 並列でReadツール実行
context_files = [
    f"{SPEC_DIR}/spec.json",
    f"{SPEC_DIR}/requirements.md",
    f"{SPEC_DIR}/design.md",
    ".claude/steering/product.md",
    ".claude/steering/tech.md",
    ".claude/steering/structure.md"
]
# Read: context_files を並列読み込み
```

**ステップ5-2: 設計分析と問題検出**

```python
# 1. 既存アーキテクチャとの整合性チェック
architecture_issues = check_architecture_consistency(design, steering)

# 2. 設計の一貫性と基準チェック
consistency_issues = check_design_consistency(design, requirements)

# 3. 拡張性と保守性の評価
maintainability_issues = check_maintainability(design)

# 4. 型安全性とインターフェース設計の評価
type_safety_issues = check_type_safety(design)

# 全問題を統合
all_issues = (architecture_issues + consistency_issues +
              maintainability_issues + type_safety_issues)

# 重大度でソートし、上位3つを抽出
critical_issues = sorted(all_issues, key=lambda x: x.severity, reverse=True)[:3]
```

**ステップ5-3: GO/NO-GO判定**

```python
# 判定基準
go_threshold = 0  # 重大な問題が0個ならGO
decision = "GO" if len(critical_issues) == 0 else "NO-GO"
```

**ステップ5-4: NO-GO時の自動修正（auto-sdd特有の処理）**

```python
if decision == "NO-GO":
    print(f"⚠️  Phase 5: 設計レビュー NO-GO判定")
    print(f"   重大な問題: {len(critical_issues)}個")

    # 問題をログに記録
    log_warning("Phase 5", f"""設計レビュー NO-GO判定
重大な問題: {len(critical_issues)}個
{format_issues(critical_issues)}
自動修正を試行します...
""")

    # 自動修正の試行
    modification_success = False

    for issue in critical_issues:
        if issue.is_auto_fixable:
            # Readツールでdesign.mdを再読込
            # Read: f"{SPEC_DIR}/design.md"

            # 問題箇所を特定して修正案を生成
            fix_suggestion = generate_fix_for_issue(issue, design_content)

            # Editツールで修正適用
            try:
                # Edit: f"{SPEC_DIR}/design.md",
                #       old_string=issue.problematic_section,
                #       new_string=fix_suggestion
                modification_success = True
                print(f"   ✅ 問題を修正: {issue.title}")
            except Exception as e:
                log_warning("Phase 5", f"自動修正失敗: {issue.title} - {e}")
        else:
            # 修正不可能な問題
            log_error("Phase 5", f"""修正不可能な重大問題を検出
問題: {issue.title}
説明: {issue.description}
推奨対応: {issue.recommendation}

設計書を手動で修正してから再実行してください。
""")

    if modification_success:
        # 再度レビュー実行
        print("   🔄 修正後の設計書を再レビュー...")
        # ステップ5-1から再実行
        # （実装上は再帰的に実行するか、ループで実装）

    decision = "GO"  # 修正完了後はGOに変更

else:
    print(f"✅ Phase 5: 設計レビュー GO判定")
```

**ログ記録**:
```python
log_phase("Phase 5", f"""設計レビュー完了
判定: {decision}
重大な問題: {len(critical_issues)}個
{"自動修正を適用" if decision == "GO" and len(critical_issues) > 0 else ""}
""")
```

**進捗報告**:
```
🔍 Phase 5: 設計書品質レビュー
✅ 設計レビュー完了
   判定: {decision}
   重大な問題: {len(critical_issues)}個
   {f"自動修正適用: {modification_count}件" if modification_count > 0 else ""}
```

---

### Phase 6: TDDフロー初期化

#### 実行内容（flow-init.mdに準拠）

**ステップ6-1: .claude/.flow/ディレクトリの準備**

```bash
# Bashツールでディレクトリ作成
mkdir -p .claude/.flow
```

**ステップ6-2: 設計書からタスク自動生成**

```python
# Readツールでdesign.mdを読み込み（すでにPhase 5で読み込み済みなら再利用）
# Read: f"{SPEC_DIR}/design.md"

# Components & Interfacesセクションから実装単位を抽出
components = extract_components_from_design(design_content)

# Testing Strategyセクションからテストタスクを生成
test_strategy = extract_testing_strategy_from_design(design_content)

# タスクリストを生成（Markdown形式）
tasks = []
task_id = 0

for component in components:
    # 各コンポーネントに対して以下のタスクを生成
    # 1. インターフェース定義
    # 2. 基本実装
    # 3. エラーハンドリング
    # 4. テストケース（test_strategyに基づく）

    tasks.append({
        "id": task_id,
        "title": f"{component.name}: インターフェース定義",
        "description": f"{component.name}の型定義とインターフェースを実装",
        "acceptance_criteria": [...],
        "estimated_lines": component.estimated_lines
    })
    task_id += 1

    tasks.append({
        "id": task_id,
        "title": f"{component.name}: 基本実装",
        "description": f"{component.name}の基本ロジックを実装",
        "acceptance_criteria": [...],
        "estimated_lines": component.estimated_lines
    })
    task_id += 1

    # ... 他のタスク

# tasks.mdの生成
tasks_md_content = format_tasks_markdown(tasks)

# Writeツールでtasks.md作成
# Write: ".claude/.flow/tasks.md", tasks_md_content
```

**ステップ6-3: state.jsonの作成**

```python
state_json_content = {
    "feature_name": FEATURE_NAME,
    "source_spec": f"{SPEC_DIR}/spec.json",
    "phase": "red",
    "current_task": 0,
    "tasks": {
        "pending": list(range(len(tasks))),
        "completed": []
    },
    "created_at": datetime.now().isoformat(),
    "updated_at": datetime.now().isoformat()
}

# Writeツールでstate.json作成
# Write: ".claude/.flow/state.json",
#        json.dumps(state_json_content, indent=2, ensure_ascii=False)
```

**ステップ6-4: 推定実行時間の算出**

```python
# タスク数から実行時間を推定
task_count = len(tasks)
estimated_minutes = task_count * 3  # 1タスク平均3分（Red+Green+Refactor）

if CODEX_ENABLED:
    estimated_minutes *= 1.5  # Codexモードは1.5倍

hours = estimated_minutes // 60
minutes = estimated_minutes % 60
```

**ログ記録**:
```python
log_phase("Phase 6", f"""TDDフロー初期化完了
パス: .claude/.flow/state.json
タスク数: {task_count}個
推定実行時間: {hours}時間{minutes}分
""")
```

**進捗報告**:
```
🔧 Phase 6: TDDフロー初期化
✅ TDDフロー初期化完了
   パス: .claude/.flow/state.json
   タスク数: {task_count}個
   推定実行時間: {hours}時間{minutes}分
   初期フェーズ: red
```

---

### Phase 7: 全タスク自動実装

#### 実行内容（flow-next.mdに準拠のループ実行）

**ステップ7-0: 初期化**

```python
step_count = 0
max_steps = 100  # 安全装置
skipped_tasks = []  # スキップしたタスクの記録
```

**ステップ7-1: メインループ**

```python
while True:
    # 安全装置チェック
    if step_count >= max_steps:
        log_warning("Phase 7", f"最大ステップ数{max_steps}に到達しました。処理を終了します。")
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
        print(f"✅ 全タスク完了！")
        break

    # タスク情報取得
    # Read: ".claude/.flow/tasks.md"
    current_task = parse_task_from_markdown(tasks_md_content, current_task_id)

    # 進捗表示
    total_tasks = len(pending_tasks) + len(completed_tasks)
    completed_count = len(completed_tasks)
    print(f"\n[ステップ {step_count}] タスク {completed_count + 1}/{total_tasks}: 「{current_task['title']}」")
    print(f"  Phase: {current_phase}")

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
            print(f"  ✅ タスク完了 (残り {len(pending_tasks) - 1}個)")

    except TaskSkippableError as e:
        # スキップ可能なエラー（テスト恒久失敗、差分規模超過など）
        log_warning("Phase 7", f"タスク{current_task_id}をスキップ: {e}")
        skipped_tasks.append({
            "task_id": current_task_id,
            "title": current_task['title'],
            "reason": str(e)
        })
        # タスクをスキップ扱いで完了
        skip_current_task(".claude/.flow/state.json", current_task_id)
        print(f"  ⚠️  タスクスキップ (残り {len(pending_tasks) - 1}個)")

    except StateInconsistencyError as e:
        # 状態不整合エラー（全体停止）
        log_error("Phase 7", f"状態不整合エラー: {e}")
```

**ステップ7-2: Red フェーズの実装**

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
        # Edit: test_file_path,
        #       old_string=<適切な挿入位置>,
        #       new_string=test_code
    else:
        # Writeツールで新規作成
        # Write: test_file_path, test_code

    # テスト実行（失敗確認）
    # Bashツール: poetry run pytest {test_file_path} -q

    if test_exit_code == 0:
        # テストが成功してしまった場合（想定外）
        raise StateInconsistencyError("Redフェーズでテストが成功しました")

    print(f"  ✅ テスト失敗を確認")
```

**ステップ7-3: Green フェーズの実装**

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

        # Editツールで修正
        # Edit: impl_file_path,
        #       old_string=<修正対象箇所>,
        #       new_string=impl_code
    else:
        # Writeツールで新規作成
        # Write: impl_file_path, impl_code

    # テスト実行（成功確認）
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # テスト失敗（想定外）
        # ロールバックを試みる
        rollback_changes(impl_file_path)
        raise TaskSkippableError("テスト失敗（Green実装後）")


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
        raise TaskSkippableError("Codex生成コードでテスト失敗")

    # クリーンアップ
    # Bashツール: rm {prompt_file} {output_file}


def apply_unified_diff(diff_content):
    """unified diff形式の差分を適用"""

    # 一時ファイルに差分を保存
    diff_file = f"/tmp/codex_diff_{datetime.now().strftime('%Y%m%d%H%M%S')}.patch"
    # Write: diff_file, diff_content

    # patchコマンドで適用
    # Bashツール: patch -p1 < {diff_file}

    if patch_exit_code != 0:
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
        #       old_string=<全内容>,
        #       new_string=content
    else:
        # Writeツールで新規作成
        # Write: file_path, content
```

**ステップ7-4: Refactor フェーズの実装**

```python
def execute_refactor_phase(task, codex_enabled):
    """Refactor フェーズ: 動作不変の品質改善"""

    print(f"  🔵 Refactor → リファクタリング中...")

    if codex_enabled:
        execute_refactor_phase_with_codex(task)
    else:
        execute_refactor_phase_normal(task)

    print(f"  ✅ テスト成功維持")


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

    # 最も安全なリファクタリングを適用
    safe_refactoring = refactor_suggestions[0]

    # Editツールで適用
    # Edit: impl_file_path,
    #       old_string=safe_refactoring.old_code,
    #       new_string=safe_refactoring.new_code

    # テスト実行（成功維持確認）
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # 失敗した場合はロールバック
        rollback_changes(impl_file_path)
        log_warning("Phase 7", f"リファクタリング失敗（ロールバック済み）")
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
        log_warning("Phase 7", f"Codex実行失敗（Refactorフェーズ）: exit code {codex_exit_code}")
        return  # リファクタリング失敗は許容

    # 出力読み込み
    # Read: output_file

    # 差分適用
    if is_unified_diff_format(codex_output):
        apply_unified_diff(codex_output)
    else:
        apply_full_content(codex_output, task)

    # テスト実行
    # Bashツール: poetry run pytest -q

    if test_exit_code != 0:
        # 失敗時はロールバック
        rollback_changes(affected_files)
        log_warning("Phase 7", f"Codexリファクタリングでテスト失敗（ロールバック済み）")

    # クリーンアップ
    # Bashツール: rm {prompt_file} {output_file}
```

**ステップ7-5: 品質管理ツールの実行**

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
        log_warning("Phase 7", f"isort failed: exit code {isort_exit_code}")
    if black_exit_code != 0:
        log_warning("Phase 7", f"black failed: exit code {black_exit_code}")
    if mypy_exit_code != 0:
        log_warning("Phase 7", f"mypy failed: exit code {mypy_exit_code}")
    if flake8_exit_code != 0:
        log_warning("Phase 7", f"flake8 failed: exit code {flake8_exit_code}")
```

**ステップ7-6: 状態更新ヘルパー関数**

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
```

**ステップ7-7: ロールバック処理**

```python
def rollback_changes(file_paths):
    """変更をロールバック（git checkoutを使用）"""

    if isinstance(file_paths, str):
        file_paths = [file_paths]

    for file_path in file_paths:
        # Bashツール: git checkout HEAD -- {file_path}
        print(f"  🔄 ロールバック: {file_path}")
```

**ログ記録**:
```python
log_phase("Phase 7", f"""全タスク自動実装完了
実行ステップ数: {step_count}
完了タスク数: {len(completed_tasks)}
スキップタスク数: {len(skipped_tasks)}
スキップ詳細:
{format_skipped_tasks(skipped_tasks)}
""")
```

**進捗報告**:
```
🔄 Phase 7: 全タスク自動実装完了
   実行ステップ数: {step_count}
   完了タスク数: {len(completed_tasks)}
   スキップタスク数: {len(skipped_tasks)}
```

---

### Phase 8: 最終品質チェック

#### 実行内容

**ステップ8-1: 全テスト実行**

```bash
# Bashツール（順次実行）:
# 1. poetry run pytest -q
# 2. poetry run pytest --cov=src --cov-report=term-missing
```

```python
# テスト実行結果の確認
if pytest_exit_code != 0:
    log_warning("Phase 8", f"テスト失敗: {pytest_output}")
else:
    print(f"✅ 全テスト成功")

# カバレッジ抽出
coverage_percentage = extract_coverage_from_output(pytest_cov_output)
print(f"✅ カバレッジ: {coverage_percentage}%")
```

**ステップ8-2: 品質管理ツール実行**

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
        log_warning("Phase 8", f"{tool}エラー")
        print(f"⚠️  {tool}エラー")
```

**ログ記録**:
```python
log_phase("Phase 8", f"""最終品質チェック完了
テスト: {"✅ 全パス" if pytest_exit_code == 0 else "⚠️ 失敗"}
カバレッジ: {coverage_percentage}%
フォーマット: {"✅" if quality_check_results["black"] else "⚠️"}
型チェック: {"✅" if quality_check_results["mypy"] else "⚠️"}
リント: {"✅" if quality_check_results["flake8"] else "⚠️"}
""")
```

**進捗報告**:
```
🔍 Phase 8: 最終品質チェック
✅ 全テスト成功
✅ フォーマット完了
✅ 型チェック完了
✅ リント完了
✅ カバレッジ: {coverage_percentage}%
```

---

### Phase 9: 自動コミット

#### 実行内容（ccommit.mdに準拠）

**ステップ9-1: 変更検出**

```bash
# Bashツール（並列実行）:
# 1. git status --porcelain
# 2. git diff --stat
```

```python
changed_files = parse_git_status(git_status_output)
diff_summary = git_diff_output

if len(changed_files) == 0:
    print("ℹ️  変更なし。コミットをスキップします。")
    return
```

**ステップ9-2: コミットメッセージ生成**

```python
# spec.jsonとdesign.mdから情報抽出（すでに読み込み済みなら再利用）
# Read: f"{SPEC_DIR}/spec.json"
# Read: f"{SPEC_DIR}/design.md"

# Conventional Commits形式でメッセージ生成
commit_type = determine_commit_type(spec_json, design_content)
# feat, fix, refactor, test, docs, etc.

scope = FEATURE_NAME

subject = f"{spec_json['description']}"

# 変更の詳細を抽出
body_lines = []
for section in ["Components & Interfaces", "Data Models", "System Flows"]:
    if section in design_content:
        summary = summarize_section(design_content, section)
        body_lines.extend(summary)

body = "\n".join(body_lines)

footer = """
🤖 Generated with Claude Code (https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
"""

commit_message = f"""{commit_type}({scope}): {subject}

{body}

{footer}"""
```

**ステップ9-3: Gitコミット実行**

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
    log_error("Phase 9", f"gitコミット失敗: {git_commit_output}")

# コミットハッシュ取得
# Bashツール: git rev-parse HEAD
commit_hash = git_rev_parse_output.strip()
```

**ステップ9-4: コミット情報の保存（オプション）**

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

## 仕様書
{SPEC_DIR}/spec.json
{SPEC_DIR}/requirements.md
{SPEC_DIR}/design.md
"""

# Bashツール: mkdir -p .claude/commits
# Writeツール: commit_info_file, commit_info_content
```

**ログ記録**:
```python
log_phase("Phase 9", f"""自動コミット完了
Message: {commit_message.split(chr(10))[0]}
Commit Hash: {commit_hash}
Files Changed: {len(changed_files)}個
""")
```

**進捗報告**:
```
📝 Phase 9: 自動コミット
✅ Gitコミット完了
   Message: {commit_message.split(chr(10))[0]}
   Commit Hash: {commit_hash}
   Files Changed: {len(changed_files)}個
```

---

### Phase 10: 最終報告

```python
# 終了時刻を記録
end_time = datetime.now()
start_time = # Phase 0で記録した開始時刻
execution_duration = end_time - start_time

# 最終報告の生成
final_report = f"""
✅ 完全自動SDD実行が正常に完了しました

【実行サマリ】
機能: {FEATURE_NAME}
説明: {spec_json['description']}
実行時間: {start_time.strftime('%H:%M:%S')} → {end_time.strftime('%H:%M:%S')} ({execution_duration})

【フェーズ別結果】
✅ Phase 1: Steering更新
✅ Phase 2: 仕様初期化 {"(スキップ)" if MODE == "existing-spec" else ""}
✅ Phase 3: 要件定義生成（要件数: {requirements_count}個）
✅ Phase 4: 設計書生成
✅ Phase 5: 設計書レビュー（判定: {decision}）
✅ Phase 6: TDDフロー初期化（タスク数: {task_count}個）
✅ Phase 7: 全タスク実装（実行ステップ: {step_count}個）
✅ Phase 8: 最終品質チェック
✅ Phase 9: 自動コミット

【実装結果】
完了タスク数: {len(completed_tasks)}個
スキップタスク数: {len(skipped_tasks)}個
実行ステップ数: {step_count}個
テストカバレッジ: {coverage_percentage}%

【品質チェック結果】
- テスト: {"✅ 全パス" if pytest_exit_code == 0 else "⚠️ 失敗"}
- フォーマット: {"✅ 完了" if quality_check_results["black"] else "⚠️ エラー"}
- 型チェック: {"✅ 完了" if quality_check_results["mypy"] else "⚠️ エラー"}
- リント: {"✅ 完了" if quality_check_results["flake8"] else "⚠️ エラー"}

【コミット結果】
Message: {commit_message.split(chr(10))[0]}
Commit Hash: {commit_hash}
Files Changed: {len(changed_files)}個

{"【警告ログ】" if len(skipped_tasks) > 0 else ""}
{format_skipped_tasks(skipped_tasks) if len(skipped_tasks) > 0 else ""}

【生成ファイル】
.claude/steering/product.md
.claude/steering/tech.md
.claude/steering/structure.md
{SPEC_DIR}/spec.json
{SPEC_DIR}/requirements.md
{SPEC_DIR}/design.md
.claude/.flow/state.json
.claude/.flow/tasks.md
[実装ファイル群: {len([f for f in changed_files if f.startswith("src/")])}個]
[テストファイル群: {len([f for f in changed_files if f.startswith("tests/")])}個]

📝 詳細ログ: {LOG_FILE}
"""

print(final_report)

# ログに記録
log_phase("Phase 10", f"""最終報告完了
実行時間: {execution_duration}
完了タスク数: {len(completed_tasks)}
スキップタスク数: {len(skipped_tasks)}
""")
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
# - Gitリポジトリでない → log_error() → exit(1)
# - 引数が空 → log_error() → exit(1)
# - 既存仕様書が存在しない → log_error() → exit(1)
```

#### 2. ファイル操作エラー（即時終了）

```python
class FileOperationError(Exception):
    """ファイル読み書きエラー"""
    pass

# 対応:
# - Read/Write/Edit失敗 → log_error() → exit(1)
```

#### 3. 設計レビューNO-GO（自動修正試行）

```python
class DesignReviewError(Exception):
    """設計レビューNG"""
    pass

# 対応:
# - 自動修正可能 → Editツールで修正 → 再レビュー
# - 自動修正不可 → log_error() → exit(1)
```

#### 4. タスク実行エラー（スキップ可能）

```python
class TaskSkippableError(Exception):
    """スキップ可能なタスクエラー"""
    pass

# 対応:
# - テスト恒久失敗 → log_warning() → skip_current_task() → 継続
# - 差分規模超過 → log_warning() → skip_current_task() → 継続
# - Codex実行失敗 → log_warning() → skip_current_task() → 継続
```

#### 5. 状態不整合エラー（即時終了）

```python
class StateInconsistencyError(Exception):
    """状態不整合エラー"""
    pass

# 対応:
# - Redフェーズでテスト成功 → log_error() → exit(1)
# - state.jsonが破損 → log_error() → exit(1)
```

#### 6. Git操作エラー（即時終了）

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
# Phase 7で100ステップ到達時は警告を出して終了
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

### 新規機能の完全自動実装

```bash
/auto-sdd "ユーザー認証機能（ログイン、ログアウト、セッション管理）"
```

### Codex連携

```bash
/auto-sdd "データ分析ダッシュボード機能" --codex
```

### 既存仕様書からの実装再開

```bash
/auto-sdd .claude/specs/user-auth/spec.json
```

### 既存仕様書からCodex連携で実装

```bash
/auto-sdd .claude/specs/user-auth/spec.json --codex
```

---

## 制限事項

- Gitリポジトリであることが必須
- 最大反復回数: 100ステップ
- タスク単位でのエラーは警告ログに記録し、継続実行
- 状態不整合エラーは全体停止
- 設計レビューでNO-GO判定時、自動修正不可能な場合は全体停止
- コミット失敗時は全体停止

---

## 想定実行時間

- **小規模機能**（タスク数: 3-5個）: 10-20分
- **中規模機能**（タスク数: 10-15個）: 30-60分
- **大規模機能**（タスク数: 20個以上）: 1-2時間

※Codexモード使用時は、API呼び出しのオーバーヘッドにより、通常の1.5-2倍の時間がかかる可能性があります。

---

## 実装チェックリスト

### Phase 0
- [ ] 8つのコマンド定義ファイルを並列読み込み
- [ ] 引数解析（--codex検出）
- [ ] モード判定（new-feature / existing-spec）
- [ ] Git/Poetry/ディレクトリ構造の確認（並列実行）
- [ ] ログファイル初期化（Writeツール）

### Phase 1
- [ ] 既存Steeringファイルを並列読み込み
- [ ] プロジェクトコンテキストを並列読み込み
- [ ] Steering 3ファイルを生成・更新（Write/Editツール）

### Phase 2
- [ ] モード分岐（existing-specはスキップ）
- [ ] 機能名生成（ケバブケース）
- [ ] 重複チェック（Bashツール）
- [ ] spec.json作成（Writeツール）

### Phase 3
- [ ] コンテキスト並列読み込み（spec.json + Steering 3ファイル）
- [ ] requirements.md生成（Writeツール）
- [ ] spec.json更新（Editツール）

### Phase 4
- [ ] コンテキスト並列読み込み（spec.json + requirements.md + Steering 3ファイル）
- [ ] Discovery & Analysis（Grep/WebSearch/WebFetch使用）
- [ ] design.md生成（Writeツール）
- [ ] spec.json更新（Editツール）

### Phase 5
- [ ] コンテキスト並列読み込み（全仕様書 + Steering）
- [ ] 設計分析と問題検出
- [ ] GO/NO-GO判定
- [ ] NO-GO時の自動修正（Editツール）

### Phase 6
- [ ] .claude/.flow/ディレクトリ作成（Bashツール）
- [ ] design.mdからタスク自動生成
- [ ] tasks.md作成（Writeツール）
- [ ] state.json作成（Writeツール）

### Phase 7
- [ ] メインループ（while True）
- [ ] state.json読み込み（各イテレーション）
- [ ] Red/Green/Refactorフェーズ実行
- [ ] 品質管理ツール実行（並列）
- [ ] 状態更新（Editツール）
- [ ] エラーハンドリング（スキップ/ロールバック）

### Phase 8
- [ ] 全テスト実行（Bashツール）
- [ ] カバレッジ取得
- [ ] 品質管理ツール実行（並列）

### Phase 9
- [ ] 変更検出（Bashツール並列）
- [ ] コミットメッセージ生成
- [ ] gitコミット実行（Bashツール）
- [ ] コミット情報保存（Writeツール）

### Phase 10
- [ ] 最終報告生成
- [ ] ログ記録

---

## バージョン履歴

- **v3.0**: 超詳細版（ツール使用の具体化、並列実行指示、状態管理詳細化、Codex連携詳細化、エラーハンドリング強化）
- **v2.0**: 詳細版（フェーズ別処理フローの詳細化）
- **v1.0**: 初版
