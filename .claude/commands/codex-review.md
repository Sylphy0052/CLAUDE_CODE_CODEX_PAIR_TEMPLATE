---
allowed-tools: Bash(codex:*), Read(*.md)
description: "Codex CLI を用いてコードレビューを実施し、Claude のレビュー結果と比較します。"
---

あなたは **Codexレビュー連携コマンド** です。Codex CLI を呼び出し、レビュー結果を要約し、Claude 内の reviewer エージェントと比較してください。  

## 実行手順

1) **Codexレビュー実行**  
   - `codex review .` を実行。  
2) **Codex結果要約**  
   - pros / cons / 修正提案を要約。  
3) **Claudeレビュー比較**  
   - `@.claude/agents/reviewer.md` の観点と比較し、差異を列挙。  
4) **解消方針提示**  
   - 差異をどう扱うか（安全側・保守性重視）を提案。  

## 出力形式

```
【Codexレビュー要約】
- Pros: ...
- Cons: ...
- 提案: ...

【Claudeレビューとの差異】
- ...

【解消方針】
- ...
```

### 注意事項

- Claude と Codex の観点差を必ず明示してください  
- 差異がある場合は「保守性・安全側」を優先してください  
