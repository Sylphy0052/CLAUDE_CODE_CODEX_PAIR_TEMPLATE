---
allowed-tools: Bash(git:*), Read(*.md)
description: "flow の状態をリセットします。引数なしは state.json のみ初期化。--hard でGit差分も破棄、--keep-tests で tests/ を保持。"
---

あなたは **リセットコマンド** です。安全性を確保しつつ、状態と作業ツリーを必要最小限で初期化します。

## 既定（引数なし）

- `.claude/flow/state.json` を初期化（バックアップ作成）
- Git 作業ツリーは変更しない

## 任意引数

- `--hard` : `git reset --hard` と `git clean -fd` を実行
- `--keep-tests` : `--hard` の際に `tests/` を除外して保持

## 実行手順

1. 既存 state を `.bak-YYYYMMDD-HHMMSS` に退避
2. 新しい state を初期値で書き込み（phase=red, tasks=[], logs=[]）
3. `--hard` が指定されていれば Git を初期化（必要なら除外パターン設定）
4. 次のアクション（flow-init/flow-next）を提案

## 出力形式

```
【リセット結果】
- state.json: 初期化 / バックアップ: .claude/flow/state.json.bak-...
- Gitリセット: 実施/未実施（--keep-tests: はい/いいえ）

【次アクション】
- flow-init で目的とタスクを再作成
- or 既存タスクが明確なら flow-next
```

## 注意

- `--hard` は **不可逆**。実行前にユーザの明示同意を要請
- 共有ブランチでの実行は避け、必ずローカルで行うこと
