# AI基本設定ファイル (CLAUDE.md)

## 1. 役割と基本規約

**役割**: SDD（仕様駆動開発）を中心としたAI駆動の自律的開発支援システム。仕様定義から設計、タスク分割、TDD実装まで一貫したワークフローを提供。Read、Write、Edit、Bashツールを直接使用してファイル操作とテスト実行を行う。

**基本規約**:

- **開発手法**: SDD (仕様駆動開発) + TDD (Red → Green → Refactor)
- **バージョン管理**: Conventional Commits (日本語)
- **言語**: Python 3.11以上
- **パッケージ管理**: Poetry（必須）
- **テストフレームワーク**: pytest
- **品質管理ツール**: mypy (型チェック), isort (import整理), black (フォーマット), flake8 (リント)
  - すべてPoetry経由で実行すること (`poetry run mypy`, `poetry run black`, etc.)

---

## 2. 行動原則

1. **仕様駆動開発 (SDD)**: すべての開発は明確な仕様書から始まる。仕様書がない場合は、まず仕様を定義してから実装に進む
2. **段階的思考**: 要件→設計→タスク→実装の順に段階的に進める
3. **承認フロー**: 各段階で人間による承認を得てから次へ進む（チーム開発時）
4. **テスト駆動 (TDD)**: 実装フェーズではコードより先にテストを書く（Red → Green → Refactor）
5. **品質重視**: 可読性・保守性・拡張性を常に考慮。すべてのコードに型ヒントを付与し、品質管理ツールを通過させる
6. **ツール直接使用**: Read/Write/Edit/Bashを直接実行（サブエージェント不使用）
7. **透明性**: 実行前に目的、実行後に結果を報告

---

## 3. 開発ワークフロー

### 3.1 標準ワークフロー（推奨）

```bash
Phase 0: プロジェクト知識管理（初回またはプロジェクト変更時）
  /steering                        → .claude/steering/*.md 作成・更新

Phase 1: 仕様定義
  /spec-init                       → .claude/specs/[feature]/spec.json 作成
  /spec-requirements               → requirements.md + 承認待ち
  /spec-design                     → design.md + 承認待ち
  /spec-tasks                      → tasks.md + 承認待ち

Phase 2: 実装フェーズへ移行
  /spec-impl [feature]             → TDD実装開始
  または
  /flow-init --from-spec [feature] → 手動でTDDサイクル管理

Phase 3: TDD実装
  /flow-next                       → Red→Green→Refactor (1ステップ)
  /flow-run                        → 自動実行（タスク完了まで）
  /flow-status                     → 進捗確認

Phase 4: 記録
  /ccommit                         → .claude/commits/ へ保存
```

### 3.2 簡易ワークフロー（小規模修正・緊急対応）

```bash
仕様書スキップモード:
  /flow-init                       → state.json + tasks.md 初期化
  /flow-run                        → Red→Green→Refactor
  /ccommit                         → コミット情報生成
```

**注意**: 簡易ワークフローは小規模修正やプロトタイプにのみ使用。中規模以上の開発では必ず標準ワークフローを使用すること。

---

## 4. コマンド体系

### 4.1 プロジェクト知識管理（Steering）

| コマンド | 説明 | 実行タイミング |
|---------|------|--------------|
| `/steering` | プロジェクト知識（product.md, tech.md, structure.md）を作成・更新 | プロジェクト開始時、大きな変更後 |
| `/steering-custom` | カスタム知識ファイルを作成（特定領域の専門知識） | 必要に応じて |

**Steeringの役割**:

- AIがプロジェクト全体のコンテキストを理解するための知識ベース
- チーム全員が共有する「プロジェクトの共通認識」
- 常にAIに読み込まれ、すべての意思決定の基準となる

### 4.2 仕様駆動開発（SDD）- **メインワークフロー**

| コマンド | 説明 | 備考 |
|---------|------|------|
| `/spec-init <description>` | 新規機能の仕様を初期化（spec.json作成） | - |
| `/spec-requirements <feature>` | 要件定義書を生成 | EARS形式 |
| `/spec-design <feature>` | 設計書を生成 | 詳細設計 |
| `/steering-custom` | カスタムSteering作成 | 特定領域の専門知識 |
| `/validate-design <feature>` | 設計書の品質レビュー | 実装前検証 |
| `/validate-gap <feature>` | 要件と実装のギャップ分析 | 実装後検証 |

### 4.3 TDD実装（flow系）- **実装フェーズのサポート**

| コマンド | 説明 | オプション |
|---------|------|-----------|
| `/flow-init <feature>` | 仕様書からTDDフロー初期化（タスク生成含む） | `--reset` |
| `/flow-status` | 進捗確認（仕様書情報含む） | - |
| `/flow-next` | TDD 1ステップ実行 | `--codex` |
| `/flow-run` | TDD自動実行 | `--steps <数>`, `--codex` |
| `/flow-reset` | フローリセット | `--hard` |

**統合機能**:

- `/flow-init` = spec-tasks機能を統合（設計書からタスク自動生成）
- `/flow-status` = spec-status機能を統合（仕様書の進捗表示）
- `/flow-run`, `/flow-next` = spec-impl機能を代替（TDD実装実行）

### 4.4 単発ユーティリティ

| コマンド | 説明 | オプション |
|---------|------|-----------|
| `/ccommit` | Conventional Commits形式のメッセージを`.claude/commits/`に生成 | `[hint]` |
| `/implement` | Greenフェーズ実装を単独実行（仕様書なし） | `--codex` |
| `/refactor` | テスト維持しながらコード改善 | `--codex` |
| `/review-codex` | コードレビュー | `--codex`, `--claude`, `--fix` |

---

## 5. 状態管理の哲学

### 5.1 仕様駆動開発における2層管理

```
【上流】仕様レイヤー (.claude/specs/)
   ↓ 承認フロー
【下流】実装レイヤー (.claude/.flow/)
```

#### `.claude/specs/[feature]/spec.json`（上流工程 - **メイン**）

- **役割**: 機能仕様の定義と承認管理
- **粒度**: 粗い（1機能全体）
- **ライフサイクル**: 長期（数日〜数週間）
- **管理項目**: 承認状態、フェーズ（initialized → requirements → design → tasks → implementation）

#### `.claude/.flow/state.json`（下流工程 - **サポート**）

- **役割**: TDD実装サイクルの管理
- **粒度**: 細かい（1タスク=数個のテストケース）
- **ライフサイクル**: 短期（数時間〜数日）
- **管理項目**: 現在フェーズ（red/green/refactor）、タスク進捗、関連仕様（source_spec）

### 5.2 データフロー（仕様駆動）

```
/spec-init
        ↓
/spec-requirements → 承認
        ↓
/spec-design → 承認
        ↓
/spec-tasks → 承認
        ↓
.claude/specs/[feature]/tasks.md
        ↓
  [/spec-impl または /flow-init --from-spec]
        ↓
.claude/.flow/tasks.md + state.json
        ↓
  [/flow-next ループ]
        ↓
実装完了
```

### 5.3 唯一の真実 (Single Source of Truth)

- **仕様フェーズ（Phase 0-2）**: `.claude/specs/[feature]/spec.json`が唯一の真実
- **実装フェーズ（Phase 3）**: `.claude/.flow/state.json`が唯一の真実
- コマンド実行前に必ず該当ファイルをReadで確認

---

## 6. 仕様駆動開発（SDD）詳細

### 6.1 3段階ワークフロー

```
Requirements（要件定義）
    ↓ 機能の目的、入出力、制約を定義
Design（設計書）
    ↓ アーキテクチャ、データ構造、API設計を定義
Tasks（タスクリスト）
    ↓ 実装の粒度、見積もり、依存関係を定義
Implementation（実装）
```

**段階的詳細化の意義**:

- 早期のスコープ明確化（手戻り防止）
- 技術的な実現可能性の事前検証
- 実装タスクの具体化

### 6.2 仕様ファイル構造

```
.claude/specs/[feature-name]/
├── spec.json           # メタデータと承認状態
├── requirements.md     # 要件定義（Phase 1で生成）
├── design.md           # 設計書（Phase 2で生成）
└── tasks.md            # タスクリスト（Phase 3で生成）
```

### 6.3 spec.json構造

```json
{
  "feature_name": "user-authentication",
  "description": "ユーザー認証機能（ログイン、ログアウト、セッション管理）",
  "created_at": "2025-10-07T10:00:00Z",
  "updated_at": "2025-10-07T12:00:00Z",
  "language": "ja",
  "phase": "design",
  "has_requirements": true,
  "has_design": true
}
```

**フェーズ一覧**:

- `initialized`: 初期化済み（spec.jsonのみ）
- `requirements`: 要件定義書生成済み
- `design`: 設計書生成済み
- `implementation`: 実装中または完了

### 6.4 仕様書の運用ルール

1. **仕様の定義**:
   - 新機能開発時は必ず仕様書を作成（`/spec-init`）
   - 要件には目的、背景、入出力、制約を明記
   - 設計にはアーキテクチャ、データ構造、API仕様を含める

2. **段階的な進行**:
   - Requirements → Design → Tasks → Implementation の順に進む
   - 各段階で内容を確認してから次へ進むことを推奨
   - 必要に応じて前の段階に戻って修正可能

3. **仕様の追跡**:
   - 実装中に仕様の曖昧さや矛盾を発見したら即座に報告
   - 仕様変更は必ず記録し、関連するテストを更新
   - spec.jsonのupdated_atを更新

---

## 7. プロジェクト知識管理（Steering）

### 7.1 Steering概念

Steeringは、AIがプロジェクト全体のコンテキストを理解するための**永続的な知識ベース**です。`.claude/steering/`ディレクトリに格納され、すべての開発フェーズで常にAIに読み込まれます。

### 7.2 標準Steeringファイル

| ファイル | 内容 | 更新タイミング |
|---------|------|--------------|
| `product.md` | プロダクト概要、主要機能、ターゲット、価値提案 | プロダクトの方向性変更時 |
| `tech.md` | 技術スタック、アーキテクチャ、開発環境、コマンド | 技術的な大きな変更時 |
| `structure.md` | ディレクトリ構造、命名規則、設計パターン | コード構造の変更時 |

### 7.3 Steeringの運用

- `/steering`で自動作成・更新（既存ファイルは保護され、新情報のみ追加）
- プロジェクト開始時に必ず実行
- 大きな技術的変更やリファクタリング後に実行推奨
- チーム全員が共有する「プロジェクトの共通言語」として機能

---

## 8. TDD実装フェーズ

### 8.1 TDDサイクル（Red→Green→Refactor）

仕様書が承認され、タスクリストが確定した後、実装フェーズに入ります。

#### Red フェーズ（失敗するテストの作成）

1. 現在タスクの受入基準から**失敗するテスト**を`tests/`配下に作成
2. `pytest -q`を実行し、テスト失敗を確認
3. フェーズを`green`に更新

#### Green フェーズ（テストを通す最小実装）

**通常モード**:

1. 失敗テストを最小修正で通す実装を作成
2. `pytest -q`で成功確認
3. 成功時にフェーズを`refactor`に更新

**Codexモード (`--codex`)**:

1. Bashで`codex` CLIを実行し、実装コードまたは差分を生成
2. unified diff形式なら差分適用、本文形式なら直接書き込み
3. `black`/`ruff`でフォーマット、`pytest -q`で検証
4. 失敗時は自動ロールバック

#### Refactor フェーズ（動作不変の品質改善）

**通常モード**:

1. 命名、重複、凝集度などを安全に改善
2. `pytest -q`で成功維持を確認
3. 成功時にタスクを完了し、次タスクに遷移（フェーズ=`red`）

### 8.2 品質管理ツール実行（必須）

コード作成・変更後は必ず以下を実行し、すべてパスすることを確認する：

```bash
# 1. Import文の整理
poetry run isort src/ tests/

# 2. コードフォーマット
poetry run black src/ tests/

# 3. 型チェック
poetry run mypy src/

# 4. リント
poetry run flake8 src/ tests/

# 5. テスト実行
poetry run pytest -q
```

**実行順序**: isort → black → mypy → flake8 → pytest

---

## 9. 実装パターン

### 9.1 テスト実行

```bash
# テスト実行（Poetry経由）
poetry run pytest -q  # 終了コード0=成功、非0=失敗

# カバレッジ付きテスト
poetry run pytest --cov=src --cov-report=term-missing
```

### 9.2 Codex連携（`--codex`時）

```bash
PROMPT_FILE=$(mktemp)
echo "$CODEX_PROMPT" > "$PROMPT_FILE"
codex --mode diff --in "$PROMPT_FILE" > output.txt
```

### 9.3 型ヒントの使用（必須）

**すべてのコードに型ヒントを付与する**

```python
# 関数の型ヒント
def calculate_price(items: list[str], tax_rate: float) -> float:
    """価格を計算する"""
    total = sum(get_item_price(item) for item in items)
    return total * (1 + tax_rate)

# Optional型の使用
from typing import Optional

def find_user(user_id: int) -> Optional[User]:
    """ユーザーを検索する。見つからない場合はNoneを返す"""
    return db.query(User).filter(User.id == user_id).first()
```

---

## 10. 使用シナリオ別推奨フロー

### シナリオA: 小規模修正・バグ修正（仕様書不要）

```bash
/flow-init
/flow-run
/ccommit
```

**所要時間**: 5-15分

### シナリオB: 新機能開発（個人作業）

```bash
# 仕様定義
/spec-init "機能説明"
/spec-tasks [feature]
# → tasks.mdレビュー・調整

# 実装
/spec-impl [feature]
/ccommit
```

**所要時間**: 1-3時間

### シナリオC: 新機能開発（チーム作業）

```bash
# プロジェクト知識更新（初回のみ）
/steering

# 完全な承認フロー
/spec-init "詳細な機能説明"
/spec-requirements [feature]  # → チームレビュー・承認
/spec-design [feature]        # → アーキテクトレビュー・承認
/spec-tasks [feature]         # → PMレビュー・承認

# 実装
/spec-impl [feature]
/ccommit
```

**所要時間**: 数日〜数週間

### シナリオD: 既存コードのリファクタリング

```bash
/refactor src/legacy_module.py --codex
/ccommit "refactor: レガシーモジュールの改善"
```

**所要時間**: 30分-1時間

---

## 11. 重要ガイドライン

### 11.1 仕様駆動開発の原則

**基本原則**: すべての開発は明確な仕様から始まる

- 仕様書なしでのコーディングは技術的負債の源泉
- 要件→設計→タスクの段階的詳細化により、実装の品質とスピードが向上
- 承認フローにより、チーム全体の認識を統一

### 11.2 状態管理

- コマンド実行前に必ず該当する状態ファイルをReadで確認
- JSON書き込みは整形形式（`indent=2`）で
- 仕様フェーズでは`spec.json`、実装フェーズでは`state.json`を参照

### 11.3 エラーハンドリング

1. エラーメッセージを分析
2. 自明な場合のみ1回だけ再試行
3. テスト失敗時はロールバック優先
4. 解決不可なら原因・試行内容・質問を報告

### 11.4 安全制約

- **変更規模上限**: 通常≤3ファイル・±120行、Codex≤±200行
- **Refactor**: テスト成功維持が絶対条件。失敗なら即ロールバック
- **型ヒント**: すべての関数・メソッドに型ヒントを付与し、mypy strictモードで検証

---

## 12. 詳細定義

各コマンドの詳細実装は以下を参照：

- `.claude/commands/kiro/` - kiro系（仕様駆動開発・メイン）
- `.claude/commands/` - flow系、単発ユーティリティ（サポート）
