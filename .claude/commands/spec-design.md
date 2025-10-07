---
description: "[設計書生成] 包括的な技術設計書を生成（cc-sdd完全版準拠）"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, WebSearch, WebFetch
argument-hint: <feature-name>
---

# 技術設計書生成

機能 **$1** の包括的な技術設計書を生成します。

## Task

### 1. コンテキスト読み込み

- `.claude/specs/$1/spec.json`: メタデータ
- `.claude/steering/`配下のすべてのファイル: プロジェクト知識
- `.claude/specs/$1/requirements.md`: 要件定義（存在する場合）

**エラーチェック**:
- `spec.json`が存在しない場合: `/spec-init`の実行を案内
- `requirements.md`が存在しない場合: `/spec-requirements`の実行を推奨（警告のみ、続行可能）

### 2. Discovery & Analysis Phase

設計書生成前に、以下の分析を実施：

#### A. 機能の分類
- **Greenfield** (新規機能): 完全な技術選定とアーキテクチャ決定
- **Extension** (既存システム拡張): 統合分析、最小限のアーキテクチャ変更
- **Simple** (CRUD、UI追加): 簡略化プロセス、確立されたパターンに従う
- **Complex** (外部システム統合、新ドメイン): 包括的な分析とリスク評価

#### B. 既存実装の分析
**既存機能の修正・拡張時は必須**:
- コードベース構造、依存関係、パターンを分析
- 再利用可能なモジュール、サービス、ユーティリティをマッピング
- ドメイン境界、レイヤー、データフローを理解
- 拡張 vs. リファクタ vs. ラップのアプローチを決定

#### C. Steeringとの整合性チェック
- コアSteering (`structure.md`, `tech.md`, `product.md`) との整合性を確認
- カスタムSteeringファイルがあれば確認
- 逸脱がある場合は根拠をドキュメント化

#### D. 技術と代替案の分析
**新機能または未知の技術領域の場合**:
- WebSearch/WebFetchを使用して最新のベストプラクティスを調査
- 関連するアーキテクチャパターンを比較（必要な場合）
- 技術スタックの代替案を評価（技術選択が必要な場合）
- 設計決定に影響を与える主要な発見をドキュメント化

#### E. 技術的リスクの評価
- パフォーマンス/スケーラビリティリスク
- セキュリティ脆弱性
- 保守性リスク
- 統合の複雑さ
- 技術的負債

### 3. 設計書の生成

以下のセクションを含む`design.md`を生成（spec.jsonの"language"フィールドで指定された言語を使用）:

#### 必須セクション

**Overview**
- Purpose（目的）
- Users（ユーザー）
- Impact（影響）
- Goals（ゴール）
- Non-Goals（非ゴール）

**Architecture**
- システムアーキテクチャの概要
- 複雑な機能の場合はMermaid図を含む

**Technology Stack and Design Decisions**
- 使用する技術スタック
- 設計決定の根拠
- コンテキストに応じて適応的に記載

**System Flows**
- Sequence/Process/Data Flowの図（Mermaid）
- 主要なユーザーフローやシステムフロー

**Components and Interfaces**
- コンポーネント構成
- インターフェース定義（型付け、`any`タイプは使用しない）

**Data Models**
- データ構造
- 明示的な型定義

**Error Handling**
- エラー処理戦略

**Testing Strategy**
- テスト戦略（単体テスト、統合テスト、E2Eテスト）

#### 条件付きセクション

**Security Considerations** (該当する場合のみ)
- 認証・認可
- データ保護
- セキュリティリスクと対策

**Performance & Scalability** (パフォーマンスクリティカルな機能の場合のみ)
- ボトルネック分析
- スケーリング戦略

**Migration Strategy** (既存システム修正の場合のみ)
- 段階的な移行計画
- バックワード互換性

### 4. spec.jsonの更新

```json
{
  "phase": "design",
  "has_design": true,
  "updated_at": "[現在のISO8601タイムスタンプ]"
}
```

## 出力

```
✅ 包括的な技術設計書を生成

📄 .claude/specs/[feature-name]/design.md
📊 セクション:
   - Overview & Goals
   - Architecture（Mermaid図含む）
   - Technology Stack & Design Decisions
   - System Flows（Mermaid図含む）
   - Components & Interfaces
   - Data Models（型定義）
   - Error Handling
   - Testing Strategy
   [該当する場合]
   - Security Considerations
   - Performance & Scalability
   - Migration Strategy

次のステップ: /flow-init [feature-name]
```

## 注意事項

### WebSearch/WebFetchの活用
- 未知の技術や最新のベストプラクティスは調査する
- 外部APIやライブラリのドキュメントを確認
- バージョン互換性を検証

### Mermaid図
- 複雑な機能では必須
- 視覚的な理解を促進
- アーキテクチャ図、シーケンス図、データフロー図を含める

### 型安全性
- すべてのインターフェースと型を明示的に定義
- `any`タイプは使用しない
- TypeScriptスタイルの型定義を推奨

### コンテキスト適応
- 機能の複雑さに応じてセクションを調整
- 不要なセクションは省略可能（条件付きセクション）
- 設計に焦点、実装コードは含めない

## エラーハンドリング

- `spec.json`が存在しない場合: `/spec-init`の実行を案内
- `$1`引数が空の場合: 機能名の入力を促す
- Steeringファイルが存在しない場合: `/steering`の実行を推奨（警告のみ、続行可能）
