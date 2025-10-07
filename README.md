# AI駆動開発ワークフロー 総合ガイド (v7.0)

このドキュメントは、本プロジェクトで定義されたカスタムAIワークフローの全体像、各コンポーネントの仕様、そして具体的な使用方法について解説する総合的なマニュアルです。

## 1. 概要と設計思想

本ワークフローは、**Claude Codeが直接ファイル操作とテスト実行を行う自律型開発システム**です。従来のサブエージェント型アーキテクチャから、**単独実行型アーキテクチャ**に完全移行し、Read/Write/Edit/Bashなどの標準ツールを使って、開発ライフサイクル全体を**統合的にサポート**します。

中核となるのは、**仕様駆動開発(SDD)とTDD実装の融合**です。明確な仕様定義から始まり、段階的詳細化を経て、TDDサイクル（Red→Green→Refactor）により高品質なコードを効率的に生産します。

### v7.0の主要な進化

- **仕様駆動開発(SDD)の統合**: 要件定義→設計→実装の一貫したワークフロー
- **プロジェクト知識管理(Steering)**: AIがプロジェクト全体を理解するための知識ベース
- **品質検証の強化**: 設計レビューと実装ギャップ分析による品質保証
- **EARS形式要件定義**: テスト可能な受入基準の標準化

## 2. ディレクトリ構成

プロジェクトは以下のディレクトリ構造を基本とします。

```plaintext
my-ai-project/
├── .claude/
│   ├── commands/              # カスタムコマンド定義
│   │   ├── steering.md
│   │   ├── steering-custom.md
│   │   ├── spec-init.md
│   │   ├── spec-requirements.md
│   │   ├── spec-design.md
│   │   ├── validate-design.md
│   │   ├── validate-gap.md
│   │   ├── flow-init.md
│   │   ├── flow-status.md
│   │   ├── flow-next.md
│   │   ├── flow-run.md
│   │   ├── flow-reset.md
│   │   ├── ccommit.md
│   │   ├── implement.md
│   │   ├── refactor.md
│   │   └── review-codex.md
│   ├── steering/              # プロジェクト知識ベース
│   │   ├── product.md
│   │   ├── tech.md
│   │   ├── structure.md
│   │   └── [custom].md       # カスタム知識ファイル
│   ├── specs/                 # 機能仕様書
│   │   └── [feature-name]/
│   │       ├── spec.json
│   │       ├── requirements.md
│   │       ├── design.md
│   │       └── tasks.md
│   ├── .flow/                 # TDD実装管理
│   │   ├── tasks.md
│   │   └── state.json
│   └── commits/               # コミット記録
│       └── YYYYMMDD_HHMMSS.md
│
├── tests/
│   └── test_*.py
│
├── src/
│   └── *.py
│
├── CLAUDE.md                  # AI基本設定ファイル
└── README.md                  # 本ファイル
```

## 3. 主要な設定ファイル

- **`CLAUDE.md`**: プロジェクト全体のAIに対する**基本設定ファイル（憲法）**。Claude Codeの役割、行動原則、コマンド体系、実装パターンを定義。
- **`.claude/commands/*.md`**: 各カスタムコマンドの詳細仕様（実行フロー、出力フォーマット、エラーハンドリング）。
- **`.claude/steering/*.md`**: プロジェクト知識ベース（プロダクト、技術、構造）。
- **`.claude/specs/[feature]/spec.json`**: 機能仕様のメタデータと承認状態を管理する**唯一の真実 (Single Source of Truth)**（上流工程）。
- **`.claude/.flow/state.json`**: TDDフェーズ（red/green/refactor）、現在タスク、完了タスク、履歴を管理する**唯一の真実 (Single Source of Truth)**（下流工程）。
- **`.claude/.flow/tasks.md`**: タスクリストをMarkdown形式で管理（`[ ]`未完了、`[x]`完了）。

## 4. 推奨ワークフロー：仕様駆動開発 + TDD実装

開発を開始する場合、以下の2つのワークフローから選択します。

### 標準ワークフロー（推奨）：中規模以上の機能開発

```bash
Phase 0: プロジェクト知識管理（初回またはプロジェクト変更時）
  /steering                        → product.md, tech.md, structure.md 作成・更新

Phase 1: 仕様定義
  /spec-init "機能説明"            → spec.json 作成
  /spec-requirements [feature]     → requirements.md 生成（EARS形式）
  /spec-design [feature]           → design.md 生成（包括的設計）
  /validate-design [feature]       → 設計書の品質レビュー（任意）
  /validate-gap [feature]          → 実装ギャップ分析（任意）

Phase 2: 実装フェーズへ移行
  /flow-init [feature]             → TDDフロー初期化（タスク自動生成）

Phase 3: TDD実装
  /flow-next                       → Red→Green→Refactor (1ステップ)
  /flow-run                        → 自動実行（タスク完了まで）
  /flow-status                     → 進捗確認

Phase 4: 記録
  /ccommit                         → コミット情報生成
```

### 簡易ワークフロー：小規模修正・緊急対応

```bash
仕様書スキップモード:
  /flow-init                       → state.json + tasks.md 初期化
  /flow-run                        → Red→Green→Refactor
  /ccommit                         → コミット情報生成
```

**注意**: 簡易ワークフローは小規模修正やプロトタイプにのみ使用。中規模以上の開発では必ず標準ワークフローを使用すること。

---

## 5. カスタムコマンド リファレンス

### 5.1. プロジェクト知識管理（Steering）

| コマンド | 説明 | 実行タイミング |
| :--- | :--- | :--- |
| **`/steering`** | **[プロジェクト知識管理]** プロジェクト全体の知識（product.md, tech.md, structure.md）を作成・更新します。 | プロジェクト開始時、大きな変更後 |
| **`/steering-custom`** | **[カスタム知識作成]** 特定領域の専門知識ファイルを作成します（例: セキュリティ、パフォーマンス）。 | 必要に応じて |

**Steeringの役割**:

- AIがプロジェクト全体のコンテキストを理解するための知識ベース
- チーム全員が共有する「プロジェクトの共通認識」
- 常にAIに読み込まれ、すべての意思決定の基準となる

---

### 5.2. 仕様駆動開発（SDD）- メインワークフロー

| コマンド | 説明 | 備考 |
| :--- | :--- | :--- |
| **`/spec-init <description>`** | **[仕様初期化]** 新規機能の仕様を初期化し、spec.jsonを作成します。 | - |
| **`/spec-requirements <feature>`** | **[要件定義生成]** EARS形式による包括的な要件定義書を生成します。 | EARS形式 |
| **`/spec-design <feature>`** | **[設計書生成]** 包括的な技術設計書を生成します（Mermaid図、型定義含む）。 | 詳細設計 |
| **`/validate-design <feature>`** | **[設計書レビュー]** 設計書の品質を対話的にレビュー・検証します。 | 実装前検証 |
| **`/validate-gap <feature>`** | **[実装ギャップ分析]** 要件と実装のギャップを分析します。 | 実装後検証 |

**EARS形式とは**: Easy Approach to Requirements Syntax。テスト可能な受入基準を記述するための標準フォーマット。

```
WHEN [イベント] THEN [システム] SHALL [応答]
IF [条件] THEN [システム] SHALL [応答]
WHILE [継続条件] THE [システム] SHALL [継続的な動作]
WHERE [場所/コンテキスト] THE [システム] SHALL [コンテキスト依存の動作]
```

---

### 5.3. TDD実装（flow系）- 実装フェーズのサポート

| コマンド | 説明 | オプション |
| :--- | :--- | :--- |
| **`/flow-init [feature]`** | **[開発フロー初期化]** `.claude/.flow`を作成し、`state.json`と`tasks.md`を初期化します。仕様書から自動でタスク生成可能。 | `--reset`（既存フロー再初期化） |
| **`/flow-status`** | **[進捗確認]** 現在のフェーズ、タスク、完了率を表示します。仕様書情報も含みます。 | `--summary`（簡易表示）、`--all`（完全履歴） |
| **`/flow-next`** | **[TDDサイクル実行]** 現在フェーズ（red/green/refactor）を検出し、1ステップ実行します。 | `--codex`（Codex CLI利用） |
| **`/flow-run`** | **[TDDサイクル連続実行]** タスク完了または指定ステップ数まで自動反復します。 | `--steps <数>`、`--codex` |
| **`/flow-reset`** | **[フローリセット]** 状態を安全にリセットし、初期状態に戻します。 | `--hard`（完全初期化・タスク進捗も破棄） |

---

### 5.4. 単発ユーティリティコマンド

| コマンド | 説明 | オプション |
| :--- | :--- | :--- |
| **`/ccommit`** | **[コミット記録生成]** 変更を解析し、Conventional Commitメッセージを生成して`.claude/commits/`に保存します。 | `[メッセージヒント]`（任意） |
| **`/implement`** | **[実装フェーズ実行]** TDDのGreenフェーズ（テストを通す実装）を単独実行します。 | `--codex` |
| **`/refactor`** | **[安全なリファクタリング]** テスト成功を維持しながらコードを改善します。失敗時は自動ロールバック。 | `--codex` |
| **`/review-codex`** | **[コードレビュー]** 指定ファイル/ディレクトリをレビューし、改善提案を生成します。 | `--codex`、`--claude`、`--fix`（修正自動適用） |

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

---

## 7. TDDサイクルの詳細

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

## 8. Codex連携の仕組み

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

## 9. エラーハンドリングと安全機能

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

## 10. 状態管理の哲学

### 2層管理システム

```
【上流】仕様レイヤー (.claude/specs/)
   ↓ 承認フロー
【下流】実装レイヤー (.claude/.flow/)
```

#### `.claude/specs/[feature]/spec.json`（上流工程 - メイン）

- **役割**: 機能仕様の定義と承認管理
- **粒度**: 粗い（1機能全体）
- **ライフサイクル**: 長期（数日〜数週間）
- **管理項目**: 承認状態、フェーズ（initialized → requirements → design → tasks → implementation）

#### `.claude/.flow/state.json`（下流工程 - サポート）

- **役割**: TDD実装サイクルの管理
- **粒度**: 細かい（1タスク=数個のテストケース）
- **ライフサイクル**: 短期（数時間〜数日）
- **管理項目**: 現在フェーズ（red/green/refactor）、タスク進捗、関連仕様（source_spec）

**唯一の真実 (Single Source of Truth)**:

- **仕様フェーズ（Phase 0-2）**: `spec.json`が唯一の真実
- **実装フェーズ（Phase 3）**: `state.json`が唯一の真実
- コマンド実行前に必ず該当ファイルをReadで確認

対話履歴（短期記憶）は一時的で不確かなため、すべての状態依存コマンドは該当ファイルを読み込んで現在の状態を把握します。

---

## 11. 使用例

### 例1: 新機能の開発（完全なSDDフロー）

```bash
# プロジェクト知識管理（初回のみ）
/steering

# 仕様定義
/spec-init "ユーザー認証機能（ログイン、ログアウト、セッション管理）"
/spec-requirements user-auth
/spec-design user-auth

# 設計品質レビュー（任意）
/validate-design user-auth

# 実装
/flow-init user-auth
/flow-run

# コミット記録生成
/ccommit "feat: ユーザー認証機能を実装"
```

---

### 例2: 小規模修正（簡易フロー）

```bash
# フロー初期化
/flow-init

# TDDサイクル自動実行
/flow-run

# コミット記録生成
/ccommit
```

---

### 例3: 既存コードのリファクタリング

```bash
# 安全なリファクタリング実行
/refactor src/utils.py "calculate_price関数の可読性を向上"

# Codexを使った高度なリファクタリング
/refactor src/core.py "process_data関数のパフォーマンスを改善" --codex

# コミット記録生成
/ccommit "refactor: コア処理のパフォーマンス最適化"
```

---

### 例4: コードレビューと改善

```bash
# Claude観点のレビュー
/review-codex src/ --claude

# Codex観点のレビュー＋自動修正
/review-codex src/utils.py --codex --fix

# コミット記録生成
/ccommit "fix: レビュー指摘事項の修正"
```

---

## 12. トラブルシューティング

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

### Q4: 仕様書が存在しない場合

```bash
# 仕様書が必要なコマンドは、存在チェックを行います
# エラーメッセージに従って、/spec-init から開始してください
/spec-init "機能説明"
```

---

## 13. 実装メモ

- **v7.0の主要変更点:**
  - 仕様駆動開発(SDD)の完全統合
  - プロジェクト知識管理(Steering)の追加
  - EARS形式要件定義の標準化
  - 設計書レビューと実装ギャップ分析の追加
  - 2層状態管理（仕様レイヤー + 実装レイヤー）

- **v6.0からの主要変更点:**
  - サブエージェント型アーキテクチャを完全削除
  - Read/Write/Edit/Bashツールによる直接実行型に移行
  - Git連携削除（コミット情報は`.claude/commits/`に保存）
  - Codex連携をBash経由のCLI実行に統一
  - 規模ガードと安全制約の強化

- **互換性:** v6.0以前のサブエージェント依存コードは動作しません。
- **拡張性:** `.claude/commands/`に新しいコマンド定義を追加することで機能拡張可能。

---

## 14. 関連ドキュメント

- [CLAUDE.md](CLAUDE.md) - AI基本設定ファイル（憲法）
- [.claude/commands/](.claude/commands/) - 各コマンドの詳細仕様

---

**ライセンス:** MIT
**バージョン:** v7.0
**最終更新:** 2025-10-07
