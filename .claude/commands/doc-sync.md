---
allowed-tools: Read(*.md), Bash(git:*), Bash(codex:*), Fetch(*)
description: "実装差分からドキュメント差分案を生成・同期します。Doc Writer と Docs Guardian のサブエージェントを活用。"
updated: "2025-09-10"
---

あなたは **ドキュメント同期コマンド (doc-sync)** です。
直近の実装変更に対し、**README/CHANGELOG/docs/** の更新案を作成し、適用可否を提示します。

## 対象

- `README.md`, `CHANGELOG.md`, `docs/**`（`docs/spec.md` は参照のみ、変更は慎重に）

## 手順

1. **差分取得**: `git diff --name-status <base>..HEAD` / `git log -1 -p`
2. **影響推定**: 公開API/CLI/設定/依存/CIに関わる変更を抽出
3. **下書き生成（Doc Writer サブエージェント）**
   - `.claude/agents/doc-writer.md` を使用し、変更点と使い方を追記案として生成
4. **乖離チェック（Docs Guardian サブエージェント）**
   - `.claude/agents/docs-guardian.md` を使用し、**実装⇔文書**の乖離を検出
5. **適用提案**: unified diff で提示し、Yes/No で適用

## Codex 連携（デフォルト）

- `codex review .` の指摘からドキュメント整合課題を抽出し、案に反映

## 引数

- `--apply` : 生成案をパッチ適用
- `--no-codex` : Codex 呼び出し抑止
- `--base <rev>` : 差分基点（既定: 上流 or `origin/<default>` 推定）

## 出力形式

【影響範囲】 API / CLI / Config / CI / 依存
【提案差分】（README/CHANGELOG/docs の unified diff）
【乖離指摘（Docs Guardian）】 上位 N 件
【適用可否】 Yes/No

### 注意事項

- **仕様 (`docs/spec.md`) の変更は最後**。先に実装との乖離を解消し、必要最低限のみ改訂
- 機密情報は**出力しない**。リンクは相対パスを優先
