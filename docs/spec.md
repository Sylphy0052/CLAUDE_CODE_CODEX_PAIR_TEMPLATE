# Codex×Claude 学習ノート作成 Webアプリ 仕様/実装計画 v1

> Notionにある「勉強ノート」を、Codex CLI（WSL）と Claude Code を並列で相互レビューしながら章立て〜Q\&A生成〜最終記事化まで自動化するウェブアプリ。

---

## 0. ゴール/非ゴール

* **ゴール**: ①3モード（Codexのみ / Claudeのみ / 相互）で記事生成、②章ごとQ\&A作成、③相互レビュー、④最終記事をNotionへ反映。ユーザは 1,2,3,5 の入力のみ。
* **非ゴール**: ノートの全文自動収集や完全自動アップロード（ユーザ確認なし）。

---

## 1. ユースケース & フロー

### 1.1 役割分担（要件に準拠）

* ユーザ: (1)参考URL添付 (2)記事の方向性入力 (3)タイトル/カテゴリ/キーワード入力 (5)章立て修正
* エージェント（本アプリがオーケストレーション）: (4)章立て提案 (6)章ごとQ\&A生成 (7)記事生成/全体確認 (8)Notionアップ

### 1.2 画面フロー（MVP）

1. **プロジェクト新規作成**（タイトル/カテゴリ/キーワード、参考URL、方向性テキスト、モード選択）
2. **章立て提案ビュー**（AI案 / 差分 / 修正フォーム）
3. **章ごとQ\&A生成ビュー**（進捗・ログ・プレビュー、Codex/Claude 出力並列比較、採用/差し戻し）
4. **全体記事プレビュー**（章マージ、要約、リンク整形）
5. **Notion送信**（ページ/データベース選択、ドラフトとしてアップロード）

```mermaid
flowchart LR
  U[ユーザ入力(1,2,3,5)] -->|作成| P[プロジェクト]
  P -->|モード選択| M[モード: codex/claude/both]
  M --> S[章立て提案]
  S --> R[ユーザ修正]
  R --> Q[章ごとQ&A生成]
  Q --> V[相互レビュー/マージ]
  V --> A[記事全体整形]
  A --> N[Notionアップロード]
```

---

## 2. モード定義

* **codex**: 章立て・Q\&A・まとめまで Codex CLI を主担当。Claude は使用しない。
* **claude**: 章立て・Q\&A・まとめまで Claude Code を主担当。Codex は使用しない。
* **both**: 章立ては Codex→Claude レビュー、Q\&Aは Codex（実装寄り）と Claude（理論寄り）を**並列生成**し、

  * 自動マージルール: 「重複除去」「用語統一」「コード→説明の順」「衝突時はユーザ選択」。

---

## 3. アーキテクチャ

### 3.1 全体像

* **Frontend**: Next.js(React) + Tailwind + shadcn/ui（または既存好みのUI）。
* **Backend API**: FastAPI (Python) + Uvicorn
* **DB**: SQLite（MVP）→ 将来 Postgres
* **ジョブ実行**: RQ or Celery + Redis（非同期キュー）
* **実行ランナー**:

  * **WSL Runner**: Codex CLI を WSL 上で実行するエージェント（REST or gRPC）
  * **Claude Runner**: Claude Code（CLI/コマンド）を呼び出すラッパー（ローカルCLI or REST）
* **同期**: WebSocket/SSE で進捗配信
* **Notion**: official Notion API（ページ/DB書き込み）

```mermaid
graph TD
  UI[Next.js Frontend] -- REST/SSE --> API[FastAPI]
  API -- enqueue --> Q[Redis Queue]
  W1[Worker: Codex] -- pull job --> Q
  W2[Worker: Claude] -- pull job --> Q
  W1 --> WSL[WSL Runner (Codex CLI)]
  W2 --> CCR[Claude Runner]
  API <---> DB[(SQLite)]
  API --> NOTION[Notion API]
```

### 3.2 ディレクトリ構成（案）

```
app/
  frontend/ (Next.js)
  api/ (FastAPI)
    models/ (SQLAlchemy)
    routers/
    services/
    workers/
    adapters/
      codex_wsl.py
      claude_code.py
      notion_client.py
    prompts/
  docker/
    docker-compose.yml
    api.Dockerfile
    frontend.Dockerfile
    worker.Dockerfile
  .env.example
```

---

## 4. データモデル（MVP）

* **Project** {id, title, category, keywords\[], direction\_text, mode, notion\_target, created\_at}
* **Reference** {id, project\_id, url, note}
* **Outline** {id, project\_id, outline\_json, status(enum: draft/approved)}
* **Chapter** {id, project\_id, index, title, status(enum), adopted\_source(enum: codex/claude/merged)}
* **QAItem** {id, chapter\_id, question, answer\_md, source(enum), raw\_log}
* **Job** {id, project\_id, type(enum: outline|chapter\_qa|merge|notion\_sync), state, payload\_json, logs}
* **Article** {id, project\_id, content\_md, toc\_json, status(enum: draft/finalized)}

> 章は Q\&A の集合として保持。Article は章をマージして生成。

---

## 5. API設計（主要）

### 5.1 プロジェクト

* `POST /projects` 新規作成（タイトル/カテゴリ/キーワード/方向性/モード/参考URL）
* `GET /projects/:id` 詳細
* `PATCH /projects/:id` 更新（章立て修正反映など）

### 5.2 章立て

* `POST /projects/:id/outline/propose` 章立て提案ジョブ起票（modeに応じてCodex/Claude/両方）
* `POST /projects/:id/outline/approve` 確定（diff付き）

### 5.3 章ごとQ\&A

* `POST /projects/:id/chapters/:idx/generate` Q\&A生成ジョブ（片方 or 両方）
* `POST /projects/:id/chapters/:idx/merge` 出力マージ
* `GET /projects/:id/chapters/:idx/preview` プレビュー取得

### 5.4 記事/Notion

* `POST /projects/:id/article/build` 全体組版
* `POST /projects/:id/notion/publish` Notionへ送信（DBレコード or Page ID 指定）

### 5.5 ジョブ/進捗

* `GET /jobs/:id` ジョブ状態
* `GET /projects/:id/stream` SSEで進捗ログ

---

## 6. ランナー/アダプタ仕様

### 6.1 Codex (WSL Runner)

* **要件**: WSL 上で `codex` コマンドを実行できる状態。Runnerは `POST /run` で `{prompt, mode}` を受け、標準出力/エラーをストリーム返却。
* **標準プロンプト雛形**

  * 章立て: 「与えられたタイトル/キーワード/方向性/参考URL を踏まえ、学習ノートの章立て（最大N章）を提案。各章の狙い/到達点を1-2文で。」
  * Q\&A: 「章タイトルと要点から、Q\&A 5-8個を作成。コード例（必要な場合）を含め Markdown で。」

### 6.2 Claude Code Runner

* **要件**: CLI もしくはローカルHTTPエンドポイントでプロンプト呼び出し可能。Codexと同じI/Fで `POST /run` を揃える。
* **役割**: 理論/背景の補強、用語統一、レビューコメント付与。

### 6.3 マージルール

1. セクション見出し一致でマージ、差異は差分表示。
2. 競合時、両案を並べ、推奨（Claude）/代替（Codex）を示す。
3. コードは Codex を優先、説明は Claude を優先。

---

## 7. プロンプト設計（ドラフト）

### 7.1 章立て提案（共通）

```
System: あなたは上級テクニカルライター兼レビュワーです。過学習せず、重複回避、初学者にもわかる構成を提案します。
User:
- タイトル: "{title}"
- カテゴリ: {category}
- キーワード: {keywords}
- 記事の方向性: {direction_text}
- 参考URL: {urls}
出力:
- 章立て(最大 {N} 章)
- 各章の狙い/到達点(1-2文)
```

### 7.2 Q\&A生成（章単位）

```
System: あなたはシニアエンジニア兼講師です。概念→手順→コード→注意点の順でQ&Aを作成します。
User:
- 章タイトル: {chapter_title}
- 章の狙い/到達点: {goal}
- モード: {codex|claude|both}
- 参考URL: {urls}
出力: Markdown(Q&A 5-8個) + 必要に応じコード
```

### 7.3 相互レビュー（both時）

```
System: あなたは文責編集者です。2つの草稿(実装寄り/理論寄り)を比較し、
重複削除、用語統一、コード→説明の順序でマージ案を1つにまとめます。
User: {codex_md} と {claude_md}
出力: 統合Markdown + 差分サマリ
```

---

## 8. UI仕様（主要画面）

### 8.1 プロジェクト作成フォーム

* 入力: タイトル/カテゴリ/キーワード、方向性テキスト（必須）、参考URL（任意）、モード選択
* 生成: Project レコード + 初回章立てジョブ起票

### 8.2 章立てレビューページ

* 左: AI案（Codex/Claude/比較）
* 右: 差分/修正フォーム（ドラッグで順序変更）
* アクション: 承認→Chapter エンティティ生成

### 8.3 章Q\&A生成ページ

* チャプターごとに「生成」ボタン
* 並列ログ（Codex/Claude）
* 「採用（codex/claude/マージ）」選択

### 8.4 全体プレビュー

* 目次（ToC）と本文
* 参照リンク整形（脚注/本文中リンク）
* Notion送信用の変換プレビュー（Notionブロック）

---

## 9. Notion連携

* **設定**: Integration Token、Database ID or Parent Page ID を .env で管理
* **実装**:

  * Page作成: タイトル/カテゴリ/キーワード/タグ/URLプロパティ
  * コンテンツ: 章見出しは `heading_2`、Q\&Aは `toggle` or `bulleted_list_item`、コードは `code`
* **アップロード動作**: Draftページを作成→ユーザ確認→最終反映（上書き or 新規）

---

## 10. 非同期実行/進捗

* すべての生成・レビューはジョブ化
* 進捗: `logs` を SSE でリアルタイム表示
* タイムアウト/再実行、リトライ（指数バックオフ）

---

## 11. セキュリティ/権限

* API Key/Token はサーバ側保管
* プロジェクト単位のRBAC（Owner/Editor/Viewer）
* 操作ログ（監査）

---

## 12. エラーハンドリング

* Codex/Claude Runner 未応答→フォールバック（片側のみで継続）
* Notion API 失敗→ローカル保存 + 再送キュー
* 章立てと本文の不一致→差分検出してユーザに要確認

---

## 13. Docker / 環境

### 13.1 docker-compose（概要）

* services: frontend, api, worker, redis
* volumes: db(./data/sqlite), cache
* env: NOTION\_TOKEN, NOTION\_DB, WSL\_RUNNER\_URL, CLAUDE\_RUNNER\_URL

### 13.2 .env.example（抜粋）

```
NOTION_TOKEN=
NOTION_DATABASE_ID=
WSL_RUNNER_URL=http://host.docker.internal:7777
CLAUDE_RUNNER_URL=http://host.docker.internal:7788
```

---

## 14. MVPスコープ

* モード: `codex` / `claude` / `both`
* 章立て→承認→章Q\&A→全体記事→Notionドラフト
* 手動で章ごと実行、並列ログ閲覧、簡易マージ

**除外（将来）**: 自動画像生成、用語集の自動抽出、引用スタイル自動最適化

---

## 15. 将来拡張

* 用語集/索引の自動生成
* 参考URLの自動要約/信頼度スコア
* リーンレビュー（PR風コメントUI）
* 複数人同時編集（CRDT）

---

## 16. リスク & 対策

* **外部CLIの不安定**: ランナー分離 + リトライ
* **Notion APIの表現限界**: Markdown→Block変換層を自前実装
* **重複/矛盾**: マージルールの明文化 + 差分UI

---

## 17. タスク分解（\~2週間MVP）

1. プロジェクト/章/QA モデルとAPI（FastAPI, SQLite）
2. Runner I/F定義とモック実装（/run）
3. 章立て提案ジョブ + 承認UI
4. 章Q\&A生成（並列）と比較UI
5. マージ＆全体プレビュー生成
6. Notionドラフト投稿
7. Docker化 & .env, README

---

## 18. サンプルAPI（擬似）

```http
POST /projects
{ "title":"Python最適化", "category":"Py", "keywords":["CUDA","メモリ"],
  "direction_text":"実装と理論の両輪", "mode":"both", "refs":[{"url":"https://..."}] }

POST /projects/{id}/outline/propose
POST /projects/{id}/outline/approve
POST /projects/{id}/chapters/1/generate  # ?side=codex|claude|both
POST /projects/{id}/chapters/1/merge
POST /projects/{id}/article/build
POST /projects/{id}/notion/publish
```

---

## 19. 最小実装の疑似コード（要点）

```python
# adapters/codex_wsl.py
async def run(prompt: str) -> str:
    return await http_post(f"{WSL_RUNNER}/run", {"prompt": prompt})

# workers/generate.py
@rq.job
def generate_chapter(project_id, chapter_idx, side):
    prompt = build_prompt(project_id, chapter_idx, side)
    if side in ("codex","both"): codex_md = run_codex(prompt)
    if side in ("claude","both"): claude_md = run_claude(prompt)
    merged = merge_outputs(codex_md, claude_md, policy="code>explain")
    save(chapter_idx, merged)
```

---

## 20. 運用

* ステージングでNotion Sandbox DBを用意
* ランナー疎通チェック（/health）
* 週次でプロンプト見直し（質・一貫性KPI）

---

### 付録A: マージ規則詳細

* 用語統一辞書（例: GPUメモリ/VRAM、誤表記矯正）
* 見出し重複の自動検出
* コードブロックの言語タグ自動補完

### 付録B: Notion変換ルール

* `#`→`heading_1`, `##`→`heading_2`
* Q\&A: `toggle`でQ:見出し / A:本文
* 脚注: 最下部にまとめ、本文に参照リンク

---

# 実装スキャフォールド v1（FastAPI + SQLite + Redis/RQ + Next.js）

> そのままコピペで最小起動できる骨組み。Codex/Claude を **並列** に呼び出し、相互レビューの土台も含みます。

## 1) ディレクトリ構成

```
app/
  docker/
    docker-compose.yml
    api.Dockerfile
    worker.Dockerfile
    frontend.Dockerfile
  api/
    requirements.txt
    main.py
    models.py
    db.py
    routers/
      projects.py
      outline.py
      notion.py
    services/
      outline_service.py
      chapter_service.py
      merge_service.py
    adapters/
      codex_wsl.py
      claude_code.py
      notion_client.py
    workers/
      queue.py
      jobs.py
      __init__.py
    prompts/
      outline_prompt.txt
      qa_prompt.txt
  frontend/
    package.json
    next.config.js
    src/
      pages/
        index.tsx
        projects/[id]/outline.tsx
        projects/[id]/chapters.tsx
      components/
        ProjectForm.tsx
        JobStream.tsx
  .env.example
  README.md
```

## 2) docker-compose & Dockerfile

```yaml
# app/docker/docker-compose.yml
version: "3.9"
services:
  redis:
    image: redis:7
    ports: ["6379:6379"]

  api:
    build:
      context: ../
      dockerfile: docker/api.Dockerfile
    env_file:
      - ../.env
    volumes:
      - ../api:/app/api
      - ../data:/app/data
    command: uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    depends_on: [redis]

  worker:
    build:
      context: ../
      dockerfile: docker/worker.Dockerfile
    env_file:
      - ../.env
    volumes:
      - ../api:/app/api
      - ../data:/app/data
    command: python -m api.workers.queue
    depends_on: [redis, api]

  frontend:
    build:
      context: ../
      dockerfile: docker/frontend.Dockerfile
    env_file:
      - ../.env
    volumes:
      - ../frontend:/app/frontend
    ports: ["3000:3000"]
    command: npm run dev --prefix /app/frontend
    depends_on: [api]
```

```dockerfile
# app/docker/api.Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY api/requirements.txt /app/
RUN pip install -r requirements.txt
COPY api /app/api
RUN mkdir -p /app/data
```

```dockerfile
# app/docker/worker.Dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY api/requirements.txt /app/
RUN pip install -r requirements.txt
COPY api /app/api
```

```dockerfile
# app/docker/frontend.Dockerfile
FROM node:20
WORKDIR /app/frontend
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend /app/frontend
```

## 3) FastAPI 主要ファイル

```txt
# app/api/requirements.txt
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.8.2
pydantic-settings==2.5.2
sqlalchemy==2.0.34
alembic==1.13.2
rq==1.16.2
redis==5.0.7
httpx==0.27.0
python-dotenv==1.0.1
```

```python
# app/api/db.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base
import os
DB_PATH = os.getenv("DB_PATH", "sqlite:////app/data/app.db")
engine = create_engine(DB_PATH, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

```python
# app/api/models.py
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey
from sqlalchemy.orm import relationship
from datetime import datetime
from .db import Base

class Project(Base):
    __tablename__ = "projects"
    id = Column(Integer, primary_key=True)
    title = Column(String, nullable=False)
    category = Column(String)
    keywords = Column(Text)  # comma-separated
    direction_text = Column(Text, nullable=False)
    mode = Column(String, default="both")
    created_at = Column(DateTime, default=datetime.utcnow)

class Outline(Base):
    __tablename__ = "outlines"
    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey("projects.id"))
    outline_json = Column(Text)
    status = Column(String, default="draft")
    project = relationship("Project")
```

```python
# app/api/main.py
from fastapi import FastAPI
from .db import engine, Base
from .routers import projects, outline
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(title="Codex×Claude Notes")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], allow_credentials=True,
    allow_methods=["*"], allow_headers=["*"],
)

Base.metadata.create_all(bind=engine)

@app.get("/health")
def health():
    return {"ok": True}

app.include_router(projects.router, prefix="/projects", tags=["projects"])
app.include_router(outline.router, prefix="/projects", tags=["outline"])
```

```python
# app/api/routers/projects.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from sqlalchemy.orm import Session
from ..db import SessionLocal
from ..models import Project

router = APIRouter()

def get_db():
    db = SessionLocal();
    try: yield db
    finally: db.close()

class ProjectIn(BaseModel):
    title: str
    category: str | None = None
    keywords: list[str] = []
    direction_text: str
    mode: str = "both"

@router.post("")
def create_project(p: ProjectIn, db: Session = Depends(get_db)):
    prj = Project(
        title=p.title, category=p.category,
        keywords=",".join(p.keywords), direction_text=p.direction_text,
        mode=p.mode,
    )
    db.add(prj); db.commit(); db.refresh(prj)
    return {"id": prj.id}

@router.get("/{pid}")
def get_project(pid: int, db: Session = Depends(get_db)):
    prj = db.get(Project, pid)
    return prj.__dict__ if prj else {"error":"not_found"}
```

```python
# app/api/routers/outline.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from sqlalchemy.orm import Session
from ..db import SessionLocal
from ..models import Outline, Project
from ..workers.jobs import enqueue_outline_job

router = APIRouter()

def get_db():
    db = SessionLocal();
    try: yield db
    finally: db.close()

class OutlineReq(BaseModel):
    max_sections: int = 6

@router.post("/{pid}/outline/propose")
def propose_outline(pid: int, body: OutlineReq, db: Session = Depends(get_db)):
    prj = db.get(Project, pid)
    if not prj: return {"error":"project_not_found"}
    job_id = enqueue_outline_job(pid, body.max_sections)
    return {"job_id": job_id}
```

## 4) RQキュー/ジョブ & ランナー（Codex/Claude）

```python
# app/api/workers/queue.py
import os
from rq import Worker, Queue, Connection
import redis
listen = ['default']
redis_url = os.getenv('REDIS_URL', 'redis://redis:6379/0')
conn = redis.from_url(redis_url)
if __name__ == '__main__':
    with Connection(conn):
        worker = Worker(list(map(Queue, listen)))
        worker.work(with_scheduler=True)
```

```python
# app/api/workers/jobs.py
import os, json
from rq import Queue
import redis
from .tasks import build_outline

redis_url = os.getenv('REDIS_URL', 'redis://redis:6379/0')
q = Queue('default', connection=redis.from_url(redis_url))

def enqueue_outline_job(project_id: int, max_sections: int):
    job = q.enqueue(build_outline, project_id, max_sections)
    return job.get_id()
```

```python
# app/api/workers/tasks.py
import httpx, os, json
from sqlalchemy.orm import Session
from ..db import SessionLocal
from ..models import Project, Outline
from ..adapters.codex_wsl import run_codex
from ..adapters.claude_code import run_claude
from ..services.merge_service import merge_outline
from ..services.outline_service import build_outline_prompt

MODE = os.getenv("DEFAULT_MODE", "both")

def build_outline(project_id: int, max_sections: int):
    db: Session = SessionLocal()
    try:
        prj = db.get(Project, project_id)
        prompt = build_outline_prompt(prj, max_sections)
        codex_md = claude_md = None
        if prj.mode in ("codex","both"):
            codex_md = run_codex(prompt)
        if prj.mode in ("claude","both"):
            claude_md = run_claude(prompt)
        merged = merge_outline(codex_md, claude_md, prefer="claude")
        outline = Outline(project_id=project_id, outline_json=json.dumps(merged), status="draft")
        db.add(outline); db.commit()
        return True
    finally:
        db.close()
```

```python
# app/api/adapters/codex_wsl.py
import os, httpx
WSL_RUNNER = os.getenv("WSL_RUNNER_URL", "http://host.docker.internal:7777")

def run_codex(prompt: str) -> str:
    try:
        r = httpx.post(f"{WSL_RUNNER}/run", json={"prompt": prompt}, timeout=120)
        r.raise_for_status(); return r.text
    except Exception as e:
        return f"# Codex Error
{e}"
```

```python
# app/api/adapters/claude_code.py
import os, httpx
CLAUDE_RUNNER = os.getenv("CLAUDE_RUNNER_URL", "http://host.docker.internal:7788")

def run_claude(prompt: str) -> str:
    try:
        r = httpx.post(f"{CLAUDE_RUNNER}/run", json={"prompt": prompt}, timeout=120)
        r.raise_for_status(); return r.text
    except Exception as e:
        return f"# Claude Error
{e}"
```

```python
# app/api/services/outline_service.py
from ..models import Project

def build_outline_prompt(prj: Project, max_sections: int) -> str:
    kws = prj.keywords.split(',') if prj.keywords else []
    return (
        "System: あなたは上級テクニカルライターです。
"
        f"User: タイトル:{prj.title}
カテゴリ:{prj.category}
キーワード:{kws}
"
        f"方向性:{prj.direction_text}
出力: 最大{max_sections}章の章立てと各章の到達点を提案。
"
    )
```

```python
# app/api/services/merge_service.py
import json

def merge_outline(codex_md: str | None, claude_md: str | None, prefer: str = "claude") -> dict:
    # MVP: 片方がなければある方を採用。両方あれば prefer を採用し、もう一方を補足に格納。
    return {
        "outline": (claude_md if prefer=="claude" and claude_md else codex_md) or codex_md or claude_md,
        "supplement": codex_md if prefer=="claude" else claude_md,
    }
```

## 5) Next.js（最小）

```json
// app/frontend/package.json
{
  "name": "notes-web",
  "private": true,
  "scripts": { "dev": "next dev" },
  "dependencies": {
    "next": "14.2.5", "react": "18.2.0", "react-dom": "18.2.0"
  }
}
```

```tsx
// app/frontend/src/pages/index.tsx
import { useState } from 'react'
export default function Home(){
  const [title,setTitle]=useState('');
  const [category,setCategory]=useState('');
  const [keywords,setKeywords]=useState('');
  const [direction,setDirection]=useState('');
  const [mode,setMode]=useState('both');
  const submit=async()=>{
    const res=await fetch('http://localhost:8000/projects',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({title: title, category, keywords: keywords.split(',').map(s=>s.trim()), direction_text: direction, mode})});
    const j=await res.json();
    if(j.id){
      await fetch(`http://localhost:8000/projects/${j.id}/outline/propose`,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({max_sections:6})});
      alert('プロジェクト作成 & 章立て提案ジョブを起票しました');
    }
  }
  return (
    <main style={{maxWidth:720,margin:'40px auto'}}>
      <h1>Codex×Claude 学習ノート</h1>
      <label>タイトル<input value={title} onChange={e=>setTitle(e.target.value)} /></label><br/>
      <label>カテゴリ<input value={category} onChange={e=>setCategory(e.target.value)} /></label><br/>
      <label>キーワード(,区切り)<input value={keywords} onChange={e=>setKeywords(e.target.value)} /></label><br/>
      <label>方向性<textarea value={direction} onChange={e=>setDirection(e.target.value)} /></label><br/>
      <label>モード<select value={mode} onChange={e=>setMode(e.target.value)}>
        <option value="codex">codex</option>
        <option value="claude">claude</option>
        <option value="both">both</option>
      </select></label><br/>
      <button onClick={submit}>作成 & 提案</button>
    </main>
  )
}
```

## 6) 並列実行の外部ランナー（ダミー）

> WSL上の Codex CLI と、Claude Code を REST 化しておく想定。最初はダミーでOK。

* **Codex Runner (port 7777)** と **Claude Runner (port 7788)** は `POST /run {prompt}` を受けてテキストを返すだけで良い。
* 実際のCLI接続は後続で差し替え。

```python
# 例: 開発時の超簡易ランナー（どちらにも流用可）
from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn
app = FastAPI()
class Req(BaseModel):
    prompt: str
@app.post('/run')
async def run(r: Req):
    return f"[DUMMY OUTPUT]
{r.prompt[:120]}..."
if __name__=='__main__':
    uvicorn.run(app, host='0.0.0.0', port=7777)
```

## 7) .env.example

```env
REDIS_URL=redis://redis:6379/0
DB_PATH=sqlite:////app/data/app.db
WSL_RUNNER_URL=http://host.docker.internal:7777
CLAUDE_RUNNER_URL=http://host.docker.internal:7788
NOTION_TOKEN=
NOTION_DATABASE_ID=
```

## 8) 起動手順（MVP）

1. `mkdir -p app/data && cd app/docker`
2. `docker compose up --build`
3. ブラウザで `http://localhost:3000` を開く
4. フォーム送信 → FastAPI が RQ ジョブを起票 → ダミーランナーが応答 → SQLite にアウトライン保存

## 9) 相互レビューの実体化（次ステップ）

* `services/merge_service.py` を高度化：

  * 章見出しの抽出・照合（正規表現）
  * 重複除去・用語辞書（VRAM/GPUメモリ等）
  * 差分UI（左右ペイン、採用/却下）
* `chapter_service.py` 追加：章ごとQ\&A生成ジョブ（codex/claude/both）

## 10) Codex/Claude 実CLI接続（例）

* Codex CLI（WSL）: ランナー側で `subprocess.run(["codex", "run", "--stdin"], input=prompt)` の標準出力を返す
* Claude Code: `claude code ...` のカスタムコマンドを同様にラップ

> これで「同じ作業を並列に走らせて相互レビュー」の**最低限の配線**が完了します。後は Q\&A生成/マージ/Notion投稿を順次追加していきます。
