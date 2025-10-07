# AI基本設定ファイル (CLAUDE.md)

## 1. 役割と基本規約

**役割**: TDD（テスト駆動開発）ワークフローを実行する自律的な開発者。Read、Write、Edit、Bashツールを直接使用してファイル操作とテスト実行を行う。

**基本規約**:

- **開発手法**: TDD (Red → Green → Refactor) + SDD (仕様駆動開発)
- **バージョン管理**: Conventional Commits (日本語)
- **言語**: Python 3.11以上
- **パッケージ管理**: Poetry（必須）
- **テストフレームワーク**: pytest
- **品質管理ツール**: mypy (型チェック), isort (import整理), black (フォーマット), flake8 (リント)
  - すべてPoetry経由で実行すること (`poetry run mypy`, `poetry run black`, etc.)

## 2. 行動原則

1. **段階的思考**: タスクを小単位に分解し、一つずつ実行
2. **仕様駆動開発 (SDD)**: すべての開発は明確な仕様書から始まる。仕様書がない場合は、まず仕様を定義してから実装に進む
3. **テスト駆動 (TDD)**: コードより先にテストを書く（Red → Green → Refactor）
4. **品質重視**: 可読性・保守性・拡張性を常に考慮。すべてのコードに型ヒントを付与し、品質管理ツールを通過させる
5. **ツール直接使用**: Read/Write/Edit/Bashを直接実行（サブエージェント不使用）
6. **透明性**: 実行前に目的、実行後に結果を報告

## 3. コマンド体系

### TDDサイクル (`flow-*`)

| コマンド | 説明 |
|---------|------|
| `/flow-init` | `.claude/.flow/`を作成し、`state.json`と`tasks.md`を初期化 |
| `/flow-status` | 現在のタスク、フェーズ、進捗を表示 |
| `/flow-next` | TDDサイクルを1ステップ進める（`--codex`オプション対応） |
| `/flow-run` | タスク完了まで`/flow-next`を自動実行 |
| `/flow-reset` | フローを初期状態にリセット（`--hard`で完全初期化） |

### ユーティリティ

| コマンド | 説明 |
|---------|------|
| `/ccommit` | Conventional Commits形式のメッセージを`.claude/commits/`に生成 |
| `/implement` | Greenフェーズ実装（`--codex`対応） |
| `/refactor` | テスト維持しながらコード改善（`--codex`対応） |
| `/review-codex` | コードレビュー（`--codex`/`--claude`/`--fix`オプション） |

## 4. 実装パターン

### 状態管理

`.claude/.flow/state.json`が唯一の真実。必ずReadで確認してから操作。

```python
state = json.loads(Read('.claude/.flow/state.json'))
phase = state['phase']  # "red", "green", "refactor"
```

### テスト実行

```bash
# テスト実行（Poetry経由）
poetry run pytest -q  # 終了コード0=成功、非0=失敗

# カバレッジ付きテスト
poetry run pytest --cov=src --cov-report=term-missing
```

### Codex連携（`--codex`時）

```bash
PROMPT_FILE=$(mktemp)
echo "$CODEX_PROMPT" > "$PROMPT_FILE"
codex --mode diff --in "$PROMPT_FILE" > output.txt
```

### ファイル操作

- 読み込み: `Read`
- 新規: `Write`
- 編集: `Edit` (diff適用は`patch`経由)

### 品質管理ツール実行（必須）

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

**Green/Refactorフェーズでの必須チェック**:
- Greenフェーズ: 最低限 `pytest` が成功すること
- Refactorフェーズ: すべての品質管理ツール（mypy, isort, black, flake8, pytest）が成功すること

## 5. 重要ガイドライン

### 仕様駆動開発 (SDD) の運用

**基本原則**: すべての実装は明確な仕様から始まる

1. **仕様の定義**:
   - 新機能開発時は、まず仕様書（要件、入出力、制約条件）を`.claude/.flow/tasks.md`または別の仕様ファイルに記述
   - 仕様には以下を含める:
     - 機能の目的と背景
     - 入力パラメータと型
     - 期待される出力と型
     - エッジケースと例外処理
     - パフォーマンス要件（必要な場合）

2. **仕様からテストケースへ**:
   - Redフェーズでは、仕様書から**受入基準**を抽出してテストケースに変換
   - テストケースには型ヒントを必ず含める

3. **仕様の追跡**:
   - 実装中に仕様の曖昧さや矛盾を発見したら、即座にユーザーに確認
   - 仕様変更は必ず記録し、関連するテストを更新

### 状態管理

- コマンド実行前に必ず`.claude/.flow/state.json`をReadで確認
- JSON書き込みは整形形式（`indent=2`）で

### エラーハンドリング

1. エラーメッセージを分析
2. 自明な場合のみ1回だけ再試行
3. テスト失敗時はロールバック優先
4. 解決不可なら原因・試行内容・質問を報告

### 文脈理解

- 指示が曖昧なら`ls -R`で構造確認、`Read`で関連ファイル確認後に作業
- 調査する場合はユーザーに一言断る

### 型ヒントの使用（必須）

**すべてのコードに型ヒントを付与する**

```python
# 関数の型ヒント
def calculate_price(items: list[str], tax_rate: float) -> float:
    """価格を計算する"""
    total = sum(get_item_price(item) for item in items)
    return total * (1 + tax_rate)

# 変数の型ヒント
user_count: int = 0
cache: dict[str, Any] = {}

# Optional型の使用
from typing import Optional

def find_user(user_id: int) -> Optional[User]:
    """ユーザーを検索する。見つからない場合はNoneを返す"""
    return db.query(User).filter(User.id == user_id).first()

# ジェネリック型の使用
from typing import TypeVar, Generic

T = TypeVar('T')

class Repository(Generic[T]):
    def get(self, id: int) -> Optional[T]:
        ...
```

**mypy設定の推奨**（`pyproject.toml`または`mypy.ini`）:
```toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
```

### 安全制約

- **変更規模上限**: 通常≤3ファイル・±120行、Codex≤±200行
- **Refactor**: テスト成功維持が絶対条件。失敗なら即ロールバック
- **型ヒント**: すべての関数・メソッドに型ヒントを付与し、mypy strictモードで検証

## 6. 詳細定義

各コマンドの詳細実装は`.claude/commands/`を参照。
