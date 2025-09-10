# HOW_TO_USE（シンプル版）

## 基本フロー

### 1. 初期化

```bash
flow-init
```

- CLAUDE.md を読み込み、目的・制約を要約  
- planner がタスクを分解  
- scaffold が最小雛形を提案  
- 最初の Red テストを提示  

---

### 2. 開発サイクル

```bash
flow-next
```

- 状態ファイル（.claude/flow/state.json）を確認  
- 自動で現在のフェーズを判定  
  - Red → 新しい失敗テストを追加  
  - Green → 最小実装を追加してテスト成功を確認  
  - Refactor → 外部挙動を変えずに整理  

---

### 3. 複数サイクル自動実行

```bash
flow-run
```

- デフォルトで 3 サイクル実行  
- テスト失敗や差分超過を検知すると停止  

---

### 4. 状態確認

```bash
flow-status
```

- 現在のフェーズ、タスク、次の候補、テスト結果を要約  

---

### 5. 状態リセット

```bash
flow-reset
```

- state.json を初期化  
- `--hard` を付けると Git 差分も破棄  

---

## オプション引数（必要な時だけ）

- `flow-next --phase green` → 明示的にフェーズ指定  
- `flow-run --max-cycles 10` → 実行サイクル数を指定  
- `flow-reset --hard` → 状態と差分を完全破棄  

---

## ポイント

- **普段は引数不要**  
- `flow-init` → `flow-next` → `flow-status` のループで進める  
- 複数まとめて進めたいときだけ `flow-run` を使う  
