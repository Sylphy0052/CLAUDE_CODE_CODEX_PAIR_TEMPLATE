# AI駆動開発ワークフロー 総合ガイド (v6.0)

このドキュメントは、本プロジェクトで定義されたカスタムAIワークフローの全体像、各コンポーネントの仕様、そして具体的な使用方法について解説する総合的なマニュアルです。

## 1. 概要と設計思想

本ワークフローは、**Claude Codeが直接ファイル操作とテスト実行を行う自律型TDD開発システム**です。従来のサブエージェント型アーキテクチャから、**単独実行型アーキテクチャ**に完全移行し、Read/Write/Edit/Bashなどの標準ツールを使って、開発ライフサイクルの中核を**自動化・サポート**します。

中核となるのは、**計画駆動のTDD実装思想**です。開発者が定義した明確なゴールに向かって、Claude Codeが自律的にTDDサイクル（Red→Green→Refactor）を回すことで、高品質なコードを効率的に生産します。

## 2. ディレクトリ構成

プロジェクトは以下のディレクトリ構造を基本とします。

```plaintext
my-ai-project/
├── .claude/
│   ├── commands/
│   │   ├── flow-init.md
│   │   ├── flow-status.md
│   │   ├── flow-next.md
│   │   ├── flow-run.md
│   │   ├── flow-reset.md
│   │   ├── ccommit.md
│   │   ├── implement.md
│   │   ├── refactor.md
│   │   └── review-codex.md
│   ├── .flow/
│   │   ├── tasks.md
│   │   └── state.json
│   └── commits/
│       └── YYYYMMDD_HHMMSS.md (自動生成)
│
├── tests/
│   └── test_*.py
│
├── src/
│   └── *.py
│
├── CLAUDE.md
└── README.md
```

## 3. 主要な設定ファイル

- **`CLAUDE.md`**: プロジェクト全体のAIに対する**基本設定ファイル（憲法）**。Claude Codeの役割、行動原則、コマンド体系、実装パターンを定義。
- **`.claude/commands/*.md`**: 各カスタムコマンドの詳細仕様（実行フロー、出力フォーマット、エラーハンドリング）。
- **`.claude/.flow/state.json`**: TDDフェーズ（red/green/refactor）、現在タスク、完了タスク、履歴を管理する**唯一の真実 (Single Source of Truth)**。
- **`.claude/.flow/tasks.md`**: タスクリストをMarkdown形式で管理（`[ ]`未完了、`[x]`完了）。

## 4. 推奨ワークフロー：計画駆動TDD開発

開発を開始する場合、以下の手順を推奨します。

### ステップ1: 開発フローの初期化 🗺️

`.claude/.flow/`ディレクトリを作成し、TDD管理用の初期ファイルを生成します。

```bash
# 初回セットアップ
/flow-init

# 既存フローの再構築（バックアップ付き）
/flow-init --reset
```

初期化されると以下のファイルが生成されます：
- `.claude/.flow/state.json`: フェーズ管理（初期値: `"phase": "red"`）
- `.claude/.flow/tasks.md`: タスクリスト（初期タスク含む）

---

### ステップ2: TDDサイクルによる実装 ⚙️

準備が整ったら、TDDサイクルを実行します。

**手動で1ステップずつ進める場合:**

```bash
# 現在フェーズに応じた処理を1回実行
/flow-next

# Codex CLIを使った実装/リファクタリング
/flow-next --codex
```

**自動で連続実行する場合:**

```bash
# 現在のタスクが完了するまで自動実行
/flow-run

# 指定ステップ数だけ実行
/flow-run --steps 3

# Codexモードで自動実行
/flow-run --codex
```

---

### ステップ3: 進捗確認 📊

現在のフェーズとタスク進捗を確認します。

```bash
# 詳細表示
/flow-status

# 簡易表示（進捗率と現在フェーズのみ）
/flow-status --summary

# 完了履歴をすべて表示
/flow-status --all
```

---

### ステップ4: コミット記録の生成 ✅

機能の実装が完了したら、変更内容を記録します。

```bash
# 変更内容を自動解析してConventional Commitメッセージを生成
/ccommit

# ヒントを与えてメッセージを調整
/ccommit "ユーザー認証のリファクタリング"
```

生成されたコミット情報は`.claude/commits/YYYYMMDD_HHMMSS.md`に保存されます。

---

## 5. カスタムコマンド リファレンス

### 5.1. Flow系コマンド（TDDサイクル管理）

| コマンド | 説明 | オプション |
| :--- | :--- | :--- |
| **`/flow-init`** | **[開発フロー初期化]** `.claude/.flow`を作成し、`state.json`と`tasks.md`を初期化します。 | `--reset`（既存フロー再初期化） |
| **`/flow-status`** | **[進捗確認]** 現在のフェーズ、タスク、完了率を表示します。 | `--summary`（簡易表示）、`--all`（完全履歴） |
| **`/flow-next`** | **[TDDサイクル実行]** 現在フェーズ（red/green/refactor）を検出し、1ステップ実行します。 | `--codex`（Codex CLI利用） |
| **`/flow-run`** | **[TDDサイクル連続実行]** タスク完了または指定ステップ数まで自動反復します。 | `--steps <数>`、`--codex` |
| **`/flow-reset`** | **[フローリセット]** 状態を安全にリセットし、初期状態に戻します。 | `--hard`（完全初期化・タスク進捗も破棄） |

---

### 5.2. 単発ユーティリティコマンド

| コマンド | 説明 | オプション |
| :--- | :--- | :--- |
| **`/ccommit`** | **[コミット記録生成]** 変更を解析し、Conventional Commitメッセージを生成して`.claude/commits/`に保存します。 | `[メッセージヒント]`（任意） |
| **`/implement`** | **[実装フェーズ実行]** TDDのGreenフェーズ（テストを通す実装）を単独実行します。 | `--codex` |
| **`/refactor`** | **[安全なリファクタリング]** テスト成功を維持しながらコードを改善します。失敗時は自動ロールバック。 | `--codex` |
| **`/review-codex`** | **[コードレビュー]** 指定ファイル/ディレクトリをレビューし、改善提案を生成します。 | `--codex`、`--claude`、`--fix`（修正自動適用） |

---

## 6. TDDサイクルの詳細

### Red フェーズ（失敗するテストの作成）

1. 現在タスクの受入基準から**失敗するテスト**を`tests/`配下に作成
2. `pytest -q`を実行し、テスト失敗を確認
3. フェーズを`green`に更新

### Green フェーズ（テストを通す最小実装）

**通常モード:**
1. 失敗テストを最小修正で通す実装を作成
2. `pytest -q`で成功確認
3. 成功時にフェーズを`refactor`に更新

**Codexモード (`--codex`):**
1. Bashで`codex` CLIを実行し、実装コードまたは差分を生成
2. unified diff形式なら差分適用、本文形式なら直接書き込み
3. `black`/`ruff`でフォーマット、`pytest -q`で検証
4. 失敗時は自動ロールバック

### Refactor フェーズ（動作不変の品質改善）

**通常モード:**
1. 命名、重複、凝集度などを安全に改善
2. `pytest -q`で成功維持を確認
3. 成功時にタスクを完了し、次タスクに遷移（フェーズ=`red`）

**Codexモード (`--codex`):**
1. Bashで`codex` CLIを実行し、リファクタリング提案を生成
2. 差分適用または本文上書き
3. `pytest -q`で動作不変を検証
4. 失敗時は自動ロールバック

---

## 7. Codex連携の仕組み

`--codex`オプションを指定すると、Bashツール経由で`codex` CLIを実行し、生成結果をファイルに反映します。

```bash
# Codexプロンプトを一時ファイルに保存
PROMPT_FILE=$(mktemp)
OUT_FILE=$(mktemp)
echo "$CODEX_PROMPT" > "$PROMPT_FILE"

# Codex CLI実行（unified diffモード優先）
codex --mode diff --in "$PROMPT_FILE" > "$OUT_FILE" 2>&1 || true

# 出力を解析:
# - unified diff形式 → そのまま差分適用
# - ファイル本文形式 → パス抽出→直接書き込み→ローカルdiff生成
```

**規模制約:**
- 通常モード: **変更ファイル≤3、総diff行数±120行**
- Codexモード: **変更ファイル≤3、総diff行数±200行**

---

## 8. エラーハンドリングと安全機能

### 自動ロールバック

- テストが失敗した場合、変更を即座にロールバック
- Refactorフェーズでは**動作不変**が絶対条件

### バックアップ機能

- `/flow-reset`実行時に`.bak`ファイルを自動生成
- `state.json`の破損検出時も自動バックアップ

### 規模ガード

- 変更規模の上限を超えそうな場合、処理を中断
- 安全な小さな変更のみを許可

---

## 9. 使用例

### 例1: 新機能の開発（TDDフロー）

```bash
# フロー初期化
/flow-init

# タスクを手動で.claude/.flow/tasks.mdに追加
# （または flow-init時に仕様書から自動生成）

# TDDサイクル自動実行
/flow-run

# コミット記録生成
/ccommit "feat: ユーザー登録機能を追加"
```

---

### 例2: 既存コードのリファクタリング

```bash
# 安全なリファクタリング実行
/refactor src/utils.py "calculate_price関数の可読性を向上"

# Codexを使った高度なリファクタリング
/refactor src/core.py "process_data関数のパフォーマンスを改善" --codex

# コミット記録生成
/ccommit "refactor: コア処理のパフォーマンス最適化"
```

---

### 例3: コードレビューと改善

```bash
# Claude観点のレビュー
/review-codex src/ --claude

# Codex観点のレビュー＋自動修正
/review-codex src/utils.py --codex --fix

# コミット記録生成
/ccommit "fix: レビュー指摘事項の修正"
```

---

## 10. 状態管理の哲学

**`.claude/.flow/state.json`は唯一の真実 (Single Source of Truth)** です。

対話履歴（短期記憶）は一時的で不確かなため、すべての状態依存コマンドは`state.json`を読み込んで現在の状態を把握します。

```json
{
  "phase": "red",
  "current_task": "ログイン処理のテストケースを作成",
  "completed_tasks": ["ユーザー登録フォームのテスト作成"],
  "pending_tasks": ["セッション維持のテスト"],
  "history": [
    {
      "timestamp": "2025-10-03T15:12:44Z",
      "phase": "green",
      "task": "ユーザー登録フォームの実装",
      "status": "success"
    }
  ]
}
```

---

## 11. トラブルシューティング

### Q1: `state.json`が破損した場合

```bash
# 安全にリセット（完了タスクは維持）
/flow-reset

# 完全初期化（すべてリセット）
/flow-reset --hard
```

### Q2: テストが通らない状態で進んでしまった場合

```bash
# 手動でstate.jsonのphaseを修正
# または /flow-reset で安全にリセット
/flow-reset
```

### Q3: Codexコマンドが見つからない場合

`--codex`オプションなしで通常モードを使用してください。Codexは環境に`codex` CLIがインストールされている場合のみ有効です。

---

## 12. 実装メモ

- **v6.0の主要変更点:**
  - サブエージェント型アーキテクチャを完全削除
  - Read/Write/Edit/Bashツールによる直接実行型に移行
  - Git連携削除（コミット情報は`.claude/commits/`に保存）
  - Codex連携をBash経由のCLI実行に統一
  - 規模ガードと安全制約の強化

- **互換性:** v5.0以前のサブエージェント依存コードは動作しません。
- **拡張性:** `.claude/commands/`に新しいコマンド定義を追加することで機能拡張可能。

---

## 13. 関連ドキュメント

- [CLAUDE.md](CLAUDE.md) - AI基本設定ファイル（憲法）
- [.claude/commands/](.claude/commands/) - 各コマンドの詳細仕様

---

**ライセンス:** MIT
**バージョン:** v6.0
**最終更新:** 2025-10-07
