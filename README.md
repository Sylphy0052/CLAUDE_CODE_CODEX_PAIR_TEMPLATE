# CLAUDE_CODE_CODEX_PAIR_TEMPLATE

このリポジトリは **Claude Code** と **Codex CLI** を組み合わせ、
**調査 → 計画 → 実装 → テスト → レビュー → 承認** の開発サイクルを自動化するテンプレートです。

## 主な特徴

- **Codex 実装 / Claude レビュー** の二重化による高信頼なコード生成
- **TDD 準拠**（Red → Green → Refactor）での実装サイクル
- サブエージェントを活用した分業（Spec Writer / Planner / Implementer / Reviewer / Doc Writer / Docs Guardian）

## コマンド概要

- **flow-init** : 初期化。spec を読み込み、調査→計画→Red テスト準備
- **flow-status** : 現在の state を要約表示
- **flow-next** : Red→Green→Refactor を **小刻みにサイクル実行（--cycles N）**
- **flow-run** : **マイルストーン or フェーズ**をまとめて進め、最後に Codex 総合確認と承認
- **flow-reset** : state とワークツリーを安全にリセット
- **tests-step** : Red→Green→Refactor を1コマンドで進める補助コマンド
- **codex-* 系** : Codex review / run / sync / bridge
- **branch-start** : 機能ブランチ作成（命名規則＆state連携）
- **doc-sync** : 実装差分からドキュメント更新案を生成・適用

## flow-next と flow-run の違い

- **flow-next** : **サイクル単位**で1歩ずつ進める
- **flow-run** : **マイルストーン or フェーズ単位**でまとめて進める

> つまり **flow-next = 小刻み**, **flow-run = まとまった実行**

## ワークフロー概要

1. **flow-init** で調査・計画を行い state.json を作成
2. 開発中は **flow-next** で少しずつ進める
3. まとまった単位では **flow-run** を用いて総合確認と承認
4. 状態は常に **flow-status** で確認
5. 不整合が起きたら **flow-reset** で巻き戻す
