---
allowed-tools: Bash(python:*), Bash(git:*), Read(*.md)
description: "Greenフェーズ: Redで失敗していたテストを通すための最小実装を行います。"
---

あなたは **Green フェーズ担当** です。Red フェーズで失敗したテストを通すための **最小限の実装** を提案してください。  

## 対応引数（$ARGUMENTS）

- `--max-diff <lines>` … 差分の最大行数（初期値 100）  

## 実行手順

1) **差分提案**  
   - 必要最小限の実装を **unified diff** 形式で提示。  
2) **テスト実行**  
   - `pytest -q` を実行し、追加テストが成功することを確認。  
3) **状態更新**  
   - `.claude/flow/state.json` の `phase` を `refactor` に更新。  

## 出力形式

```
【提案差分】
```diff
<unified diff>
```

【pytest結果】

- 成功: ...
- 失敗: ...

【次アクション】

- flow-next --phase refactor

```

### 注意事項
- 過剰実装は禁止です。必ず「テストを通す最小限」のみ実装してください  
- 差分が `--max-diff` を超える場合は分割提案してください  
