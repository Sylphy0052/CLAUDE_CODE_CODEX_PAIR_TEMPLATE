# HOW_TO_USE

このマニュアルは **日常の使い方** に特化しています。
README.md が「全体設計の説明」なのに対し、こちらは「具体的な操作手順」を示します。

---

## 初回セットアップ

```bash
claude flow-init
```

- `docs/spec.md` を読み込み、Spec Writer と Planner が調査・計画を行う
- 最初の Red テストを生成し、`.claude/flow/state.json` を作成

---

## 小刻みに進める（flow-next）

```bash
claude flow-next --cycles 1
```

- Red → Green → Refactor の **1サイクルだけ** 実行
- `--cycles N` で Nサイクルまとめて進めることも可能
- 内部で `tests-step` / `codex-review` を実行し、Codex と Reviewer の結果を突き合わせる
- 公開API/設定変更があれば `doc-sync`（提案）を起動して更新案を提示

---

## まとめて進める（flow-run）

```bash
claude flow-run --milestone current
```

- **1マイルストーン完了まで** flow-next を繰り返す
- 最後に `codex-run` / `codex-bridge` を使い総合確認し、必要に応じて `doc-sync --apply` で文書整合を確保
- Yes/No 確認の上で承認・統合

例: フェーズ単位で進める場合

```bash
claude flow-run --phase green
```

---

## 状態確認

```bash
claude flow-status
```

- 現在の phase / task / next_tasks を確認

---

## リセット

```bash
claude flow-reset --hard
```

- state とワークツリーを安全に復元

---

## 注意点

- **flow-next は開発者が小刻みに進める用**
- **flow-run はまとまった単位で総合確認する用**
- Codex 連携はデフォルト有効（抑止する場合は `--no-codex`）
