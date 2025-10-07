---
description: "[要件定義生成] EARS形式による包括的な要件定義書を生成"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: <feature-name>
---

# 要件定義生成

機能 **$1** の包括的な要件定義書をEARS形式で生成します。

## Task

### 1. コンテキスト読み込み

#### Steeringコンテキスト
- アーキテクチャ: `.claude/steering/structure.md`
- 技術制約: `.claude/steering/tech.md`
- プロダクト: `.claude/steering/product.md`
- カスタムSteering: `.claude/steering/`配下のすべてのファイル

#### 既存Specコンテキスト
- spec.json: `.claude/specs/$1/spec.json`（説明とメタデータ）
- 現在のディレクトリ内容: `ls -la .claude/specs/$1/`

**エラーチェック**: spec.jsonが存在しない場合は`/spec-init`の実行を案内

### 2. 初期要件の生成

`spec.json`の`description`フィールドから初期要件を生成し、ユーザーとの反復で洗練します。

**生成ガイドライン**:
1. **コア機能に集中**: ユーザーのアイデアから本質的な機能を抽出
2. **EARS形式を使用**: すべての受入基準に適切なEARS構文を適用
3. **逐次質問なし**: 最初に初期版を生成し、ユーザーフィードバックで反復
4. **管理可能な範囲**: ユーザーレビューで拡張可能な堅実な基盤を作成
5. **適切な主語を選択**: ソフトウェアプロジェクトでは具体的なシステム/サービス名を使用（例: 「認証システム」）

### 3. EARS形式要件

**EARS (Easy Approach to Requirements Syntax)** は受入基準の推奨フォーマット:

**主要EARSパターン**:
- `WHEN [イベント/条件] THEN [システム/主語] SHALL [応答]`
- `IF [前提条件/状態] THEN [システム/主語] SHALL [応答]`
- `WHILE [継続条件] THE [システム/主語] SHALL [継続的な動作]`
- `WHERE [場所/コンテキスト/トリガー] THE [システム/主語] SHALL [コンテキスト依存の動作]`

**複合パターン**:
- `WHEN [イベント] AND [追加条件] THEN [システム/主語] SHALL [応答]`
- `IF [条件] AND [追加条件] THEN [システム/主語] SHALL [応答]`

### 4. 要件定義書の構造

`requirements.md`を以下の構造で生成（spec.jsonの"language"フィールドで指定された言語を使用）:

```markdown
# 要件定義書

## はじめに
[機能とそのビジネス価値を要約した明確な導入]

## 要件

### 要件1: [主要目的領域]
**目的**: [役割/ステークホルダー]として、[機能/能力/成果]を望む。なぜなら[利益]

#### 受入基準
1. WHEN [イベント] THEN [システム/主語] SHALL [応答]
2. IF [前提条件] THEN [システム/主語] SHALL [応答]
3. WHILE [継続条件] THE [システム/主語] SHALL [継続的な動作]
4. WHERE [場所/コンテキスト/トリガー] THE [システム/主語] SHALL [コンテキスト依存の動作]

### 要件2: [次の主要目的領域]
**目的**: [役割/ステークホルダー]として、[機能/能力/成果]を望む。なぜなら[利益]

#### 受入基準
1. WHEN [イベント] THEN [システム/主語] SHALL [応答]
2. WHEN [イベント] AND [条件] THEN [システム/主語] SHALL [応答]

### 要件3: [追加の主要領域]
[すべての主要機能領域についてパターンを継続]
```

### 5. メタデータの更新

`spec.json`を更新:

```json
{
  "phase": "requirements-generated",
  "has_requirements": true,
  "updated_at": "[現在のISO8601タイムスタンプ]"
}
```

### 6. ドキュメント生成のみ

要件定義書の内容のみを生成。実際のドキュメントファイルにはレビューや承認の指示を含めない。

## 出力

```
✅ 要件定義書を生成

📄 .claude/specs/[feature-name]/requirements.md
📋 要件数: [N]個
📝 受入基準: [M]項目（EARS形式）

次のステップ: /spec-design [feature-name]
```

## エラーハンドリング

- `spec.json`が存在しない場合: `/spec-init`の実行を案内
- `$1`引数が空の場合: 機能名の入力を促す
- Steeringファイルが存在しない場合: `/steering`の実行を推奨（警告のみ、続行可能）

## 注意事項

- 実装の詳細には焦点を当てず、要件（何を実現するか）に集中する
- EARS形式の受入基準は、後でテストケースに変換可能な形式で記述する
- 初期版を生成後、ユーザーとの対話で洗練する
