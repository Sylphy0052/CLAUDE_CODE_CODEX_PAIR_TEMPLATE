---
description: "Claude Code の運用規約と TDD ワークフロー定義"
---

# CLAUDE.md

## 目的

- このプロジェクトでは **TDD（Red→Green→Refactor）** を徹底します  
- **Claude Code** と **Codex CLI** を並行利用し、相互レビューで品質を高めます  
- すべてのコード変更は **パッチ提案→承認→適用** の手順を必須とします  

## 原則

1. 最小実装：テストで要求された範囲のみ実装  
2. 明示的差分：常に unified diff で提案  
3. 可逆性：ロールバック手順を含める  
4. 自動化：テスト・Lint・型チェックを自動実行  
5. 相互レビュー：Claude↔Codex 双方で検証  

## ワークフロー

1. flow-init … 目標読込・タスク分解・Redテスト計画  
2. tests-red … 失敗テストを追加  
3. tests-green … 最小実装で成功させる  
4. tests-refactor … 外部挙動を変えず整理  
5. reviewer / codex-review … 相互レビュー  
6. flow-next … 次タスクへ継続  

## 役割

- planner … 要件を小タスク化  
- tdd-coach … TDDの順守を強制  
- reviewer … 設計/境界値/セキュリティ等のレビュー  
- scaffold … 雛形・最小構成生成  
- spec-writer … 要件をPRD化  
- doc-writer … ドキュメント更新  
- codex-bridge … Codex実行結果の要約と比較  

## 出力形式

- 差分 (unified diff)  
- 変更理由・影響範囲・ロールバック手順  
- テスト結果 (pytest / codex run tests)  
