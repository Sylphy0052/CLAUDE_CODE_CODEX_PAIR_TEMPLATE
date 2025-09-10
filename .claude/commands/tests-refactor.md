---
allowed-tools: Bash(python:*), Bash(git:*), Read(*.md)
description: "Refactorフェーズ: 外部挙動を変えずにコードを整理し、テストが成功することを確認します。"
---

あなたは **Refactor フェーズ担当** です。Green フェーズで通ったテストを保持したまま、コードを整理してください。  

## 実行手順

1) **リファクタリング提案**  
   - 命名整理・重複排除・関数分割などを **unified diff** で提示。  
2) **テスト実行**  
   - `pytest -q` を再実行し、全テストが成功していることを確認。  
3) **状態更新**  
   - `.claude/flow/state.json` の `phase` を `red` に戻し、次のタスクへ進行可能に。  

## 出力形式

```
【提案差分】
```diff
<unified diff>
```

【pytest結果】

- 全テスト成功: Yes/No

【次アクション】

- flow-next --phase red

```

### 注意事項
- 外部挙動を一切変更してはいけません  
- リファクタリングの根拠を簡潔に説明してください  
