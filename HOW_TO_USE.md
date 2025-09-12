# HOW_TO_USE

このマニュアルは **日常の使い方（具体的な操作手順）** に特化しています。
`README.md` が「全体設計の説明」なのに対し、こちらは「毎日の運用で実際に叩くコマンド」をまとめます。

対象コマンドは以下の5つのみです（`.claude/commands/` に含まれるもの）:

- `flow-init`
- `flow-next`
- `flow-run`
- `flow-status`
- `flow-reset`

> ここに記載のないコマンド（例: `branch-start`, `commit-feature`, `codex-run` など）は本構成には **存在しません**。

---

## 初回セットアップ（flow-init）

```bash
claude flow-init
```

- `docs/spec.md` を読み、**調査 → 計画 → タスク分割** を実施
- `.claude/flow/state.json` / `.claude/flow/tasks.json` /（必要に応じて）設定ファイルを生成

オプション:

- `--spec "<file>"` : 仕様ファイルの明示指定（既定: `docs/spec.md`）
- `--goal "<text>"` : 目的テキストを直接指定（spec より優先）
- `--reset` : 既存 state を無視して再生成（古い state は `.bak` に退避）

---

## 小刻みに進める（flow-next）

```bash
claude flow-next --cycles 1
```

- **TDD サイクル（Red → Green → Refactor）** を 1 回実行（`--cycles N` で複数回）
- フェーズは自動判定。必要なら明示上書き可能

主なオプション:

- `--cycles <N>` : サイクル回数（既定: 1）
- `--phase <red|green|refactor>` : 単一フェーズのみ実行
- `--max-diff <lines>` : 差分表示/適用の上限（既定: 200）
- `--no-codex` : Codex 呼び出しを抑止（Claude 単独範囲のみ）
- `--dry-run` : 変更せず提案のみ
- `--wip-allow` : `.claude/flow/config.json` の `wip_limit` を無視
- `--format <md|json>` : 出力形式（既定: `md`）
- `--compact` / `--verbose` : 表示の簡易化 / 詳細化

停止・安全策（代表例）:

- `--max-diff` 超過、DAG 不整合、Red が失敗しない / Green が成功しない、破壊的変更の兆候 等で停止し要約提示

---

## まとめて進める（flow-run）

```bash
# いずれか必須: --milestone または --phase
claude flow-run --milestone current
# 例: 特定フェーズをまとめて
claude flow-run --phase green
```

- 内部で `flow-next --cycles 1` を反復し、**完了条件**（マイルストーン or フェーズ）まで進行
- 完了後に **総合テスト → 相互レビュー（Claude & Codex） → 人間承認** を実施

主なオプション:

- `--milestone "<name|index|current>"` : マイルストーンを完了まで反復
- `--phase <red|green|refactor>` : 指定フェーズを完了まで反復
- `--max-steps <M>` : 内部反復の上限（既定: 10）
- `--no-codex` : Codex 呼び出し抑止
- `--dry-run` : 適用せず結果のみ表示
- `--confirm`（既定） / `--yes` / `--no-input` : 承認フロー（対話/非対話）制御
- `--format <md|json>` / `--compact` / `--verbose`

停止・安全策は `flow-next` と同様（大規模差分や DAG 不整合などで中断し要約提示）。

---

## 状態確認（flow-status）

```bash
claude flow-status
```

- 現在の **phase / current_task / next_tasks / last_diff** 等を要約表示
- 未初期化なら `flow-init` 実行を案内

オプション:

- `--compact` : 簡易表示
- `--format <md|json>` : 出力形式（既定: `md`）
- `--spec "<file>"` : アンカー出力等で参照する仕様ファイルを指定（必要時）

表示ルール（抜粋）:

- 直近の差分がある場合のみ ```diff ブロックを表示
- `phase=planned` は `last_test_result` を非表示
- 詳細な工程は `flow-run` を参照

---

## 安全に巻き戻す（flow-reset）

```bash
# 直前バックアップから復元（確認プロンプトあり）
claude flow-reset
# 指定バックアップから強制的に復元
claude flow-reset --from .claude/flow/state.json.bak-20250912 --hard --yes
```

- 作業ツリーと state を **安全に巻き戻す**（Codex 連携なし）
- 未コミット変更は退避し、必要なら直前のバックアップへ復元

主なオプション:

- `--from <path>` : 指定バックアップから復元（なければ直近の候補）
- `--hard` : 退避後に作業ツリーをクリーン化（ローカル変更を除外）
- `--keep-worktree` : ワークツリーは維持して **state のみ** 復元
- `--confirm`（既定） / `--yes` / `--no-input`
- `--format <md|json>` / `--compact` / `--dry-run`

注意:

- バックアップ作成に失敗した場合は中止
- 復元後は必ず `flow-status` で整合確認

---

## 典型フロー（例）

```bash
# 1) 初期化
claude flow-init --spec docs/spec.md

# 2) 小刻みに前進（TDD を回す）
claude flow-next --cycles 1
claude flow-status

# 3) マイルストーン単位でまとめて進める
claude flow-run --milestone current --max-steps 10 --confirm

# 4) 何かおかしい場合は安全に巻き戻す
claude flow-reset --dry-run    # 影響範囲をまず確認
```

---

## 生成/参照される主なファイル

- `.claude/flow/state.json`（必須・現在地）
- `.claude/flow/tasks.json`（DAG 検証・アンカー生成）
- `docs/spec.md` / `docs/tasks.md`（仕様とタスク定義・アンカー出力）

---

## 推奨ワークフロー

| 目的 | コマンド | 備考 |
|------|----------|------|
| 新規プロジェクトを開始する | `claude flow-init --spec docs/spec.md` | 初期状態（phase=planned）を生成 |
| 1サイクルだけ TDD を回す | `claude flow-next --cycles 1` | Red → Green → Refactor を1回実行 |
| 複数サイクルをまとめて進める | `claude flow-next --cycles 3` | 小刻みに進めつつ効率化したい場合 |
| マイルストーンを一気に進める | `claude flow-run --milestone current` | 完了後に総合テスト・相互レビュー・承認 |
| 状態を確認する | `claude flow-status` | 現在の phase / current_task / next_tasks を表示 |
| 異常時ややり直したい場合 | `claude flow-reset --from <backup>` | 直近バックアップから安全に復元 |

### 運用のヒント

- 開発中は **`flow-next` をメイン**に使い、小さく進める
- 節目で **`flow-run` を使ってマイルストーン単位で検証・承認**
- CI/CD では `--no-input --format json` を組み合わせ、自動検証に活用
- 重大な不整合が起きた場合は **`flow-reset`** を最初に試す

---
