# Open WebUI 项目分析文档

> 生成时间：2026-08-17 · 版本：v0.11.0

## 1. 项目概述

Open WebUI 是一个**可扩展、功能丰富、用户友好的自托管 AI 平台**，可完全离线运行。它支持 Ollama、OpenAI 兼容 API 等多种 LLM 运行时，内置 RAG（检索增强生成）推理引擎，是一个强大的 AI 部署方案。

本仓库为 Open WebUI 的完整源码仓库（开发模式），包含 Python 后端与 SvelteKit 前端，可本地从源码启动运行。

## 2. 技术栈

| 层级 | 技术 |
| --- | --- |
| 后端 | Python 3.12、FastAPI、Uvicorn、SQLAlchemy (async)、Alembic |
| 前端 | SvelteKit (Svelte 5)、Vite、TypeScript、TailwindCSS 4 |
| 数据库 | SQLite（默认，aiosqlite）/ PostgreSQL（psycopg，可选） |
| 向量库 | ChromaDB（默认）、PGVector、Qdrant、Milvus、Elasticsearch、OpenSearch 等 9 种 |
| 实时通信 | Socket.IO、WebSocket（Redis 支撑多节点） |
| LLM 接入 | Ollama、OpenAI 兼容 API、Anthropic、Google Gemini 等 |
| 其他 | Redis（会话/缓存）、OpenTelemetry（可观测）、Pyodide（浏览器内 Python） |

## 3. 目录结构

```
open-webui/
├── backend/                     # Python 后端
│   ├── open_webui/
│   │   ├── main.py              # FastAPI 应用入口
│   │   ├── config.py            # 运行时配置（动态）
│   │   ├── env.py               # 环境变量加载与校验
│   │   ├── events.py            # 事件系统
│   │   ├── tasks.py             # 后台定时任务
│   │   ├── routers/             # API 路由（auths、chats、models、retrieval 等 30+）
│   │   ├── models/              # 数据模型（chats、users、channels、memory 等 26 个）
│   │   ├── retrieval/           # RAG：向量库、文档加载器、Web 搜索
│   │   ├── migrations/          # Alembic 数据库迁移（60+ 版本）
│   │   ├── utils/               # 工具模块（auth、files、chat 等）
│   │   └── socket/              # Socket.IO 实时通信
│   ├── requirements.txt         # 后端依赖（完整）
│   ├── start.sh / start_windows.bat / dev.sh   # 启动脚本
│   └── .webui_secret_key        # 首次启动自动生成
├── src/                         # 前端（SvelteKit）
│   ├── routes/                  # 页面路由（首页、认证、watch）
│   └── lib/
│       ├── components/          # UI 组件（chat、admin、channel、calendar 等）
│       ├── apis/                # 前端 API 客户端（audio、auths、chats、models 等）
│       ├── stores/              # 状态管理（chatList 等）
│       └── utils/               # 工具（codemirror、pptxToHtml 等）
├── static/                      # 静态资源（emoji、favicon 等）
├── docs/                        # 项目文档
├── Dockerfile / docker-compose*.yaml   # 容器化部署
├── package.json                 # 前端依赖与脚本
└── pyproject.toml               # Python 包定义
```

## 4. 核心功能

| 功能 | 说明 |
| --- | --- |
| 多模型对话 | 同时接入 Ollama + 任意 OpenAI 兼容 API，多模型并行响应 |
| RAG 知识库 | 9 种向量库 + 混合检索（BM25 + 向量）+ 重排序，`#` 命令加载文档 |
| Web 搜索 | SearXNG、Google PSE、Brave、Tavily 等 30+ 搜索服务商 |
| 插件机制 | Filters、Actions、Pipes、Tools、Skills，支持 MCP/OpenAPI 工具服务器 |
| 多用户与权限 | RBAC 角色/用户组权限控制，LDAP/OAuth/SSO/SCIM |
| 记忆系统 | AI 跨会话记住用户事实 |
| 频道与自动化 | 实时共享频道（Channels）、定时自动化（Automations） |
| 语音/视频 | Whisper 本地语音转写、多 TTS 引擎 |
| 图片生成 | DALL·E、Gemini、ComfyUI、AUTOMATIC1111 |
| 管理分析 | 用量统计、Token 消耗、成本追踪、模型评测 Arena/ELO |

## 5. 启动方式

### 5.1 本地开发启动（推荐）

```bash
# 1) 后端
cd backend
python -m venv .venv
.venv\Scripts\pip install -r requirements.txt
# 启动（Windows）
.venv\Scripts\activate
set CORS_ALLOW_ORIGIN=http://localhost:5173;http://localhost:8080
uvicorn open_webui.main:app --port 8080 --host 0.0.0.0 --reload

# 2) 前端（另开终端）
npm install
npm run dev          # Vite dev server → http://localhost:5173
```

> 注意：项目 `.npmrc` 设置了 `engine-strict=true`，Node 版本需在 18.13–22.x 之间；若使用更高版本 Node，需 `npm install --engine-strict=false`。

### 5.2 生产启动

```bash
# 后端（打包前端后 FastAPI 同端口提供页面）
cd backend && python start.py   # 或 start.sh / start_windows.bat
# 访问 http://localhost:8080
```

### 5.3 Docker 启动

```bash
docker run -d -p 3000:8080 -v open-webui:/app/backend/data \
  --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

## 6. 关键配置（环境变量）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | 8080 | 后端监听端口 |
| `HOST` | 0.0.0.0 | 监听地址 |
| `WEBUI_SECRET_KEY` | 自动生成 | 会话/JWT 密钥（存 `.webui_secret_key`） |
| `OLLAMA_BASE_URL` | http://localhost:11434 | Ollama 服务地址 |
| `OPENAI_API_BASE_URL` / `OPENAI_API_KEY` | 空 | OpenAI 兼容 API |
| `CORS_ALLOW_ORIGIN` | `*` | CORS 白名单（开发需含 5173/8080） |
| `ENABLE_PLUGINS` | true | 启用 Tools/Functions |
| `WEBUI_URL` | — | 对外访问地址 |
| `DATABASE_URL` | sqlite:///webui.db | 切换 PostgreSQL 等 |

配置加载优先级：环境变量 → 根目录 `.env` 文件 → 默认值（`backend/open_webui/env.py`）。

## 7. 数据存储

- **数据库**：`backend/data/webui.db`（SQLite），包含用户、聊天、消息、知识库、频道、记忆等 30+ 张表，由 Alembic 管理迁移。
- **上传文件**：`backend/data/uploads/`。
- **向量库**：默认 ChromaDB，数据位于 `backend/data/chroma`；可切换 PGVector/Milvus/Qdrant 等。

## 8. 前后端通信

- 前端通过 `src/lib/apis/` 下的客户端调用 `/api/v1/*` REST 接口（含 SSE 流式响应）。
- 聊天流式输出、频道实时消息走 Socket.IO（`backend/open_webui/socket/`）。
- 开发模式下 Vite 代理将 `/api`、`/ollama` 等转发到后端 8080。

## 9. 扩展与二次开发入口

| 需求 | 入口 |
| --- | --- |
| 新增 API | `backend/open_webui/routers/` 新增路由并注册到 `main.py` |
| 数据模型/迁移 | `backend/open_webui/models/` + `migrations/versions/` |
| 前端页面 | `src/routes/` + `src/lib/components/` |
| 自定义工具/技能 | 后端 `tools/`、前端「工作区」管理界面 |
| RAG 改造 | `backend/open_webui/retrieval/`（向量库、加载器、搜索） |
