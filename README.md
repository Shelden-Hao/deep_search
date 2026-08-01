# DeepSearch 深度智能搜索与报告生成平台

<div align="center">

**基于 LangGraph + DeepAgents 的企业级多智能体深度搜索系统**

协调「网络搜索 / 数据库查询 / 知识库问答」三大专家助手，一站式完成**信息检索 → 知识融合 → 文档生成**的完整闭环。

![Python](https://img.shields.io/badge/Python-3.13-blue) ![FastAPI](https://img.shields.io/badge/FastAPI-0.129-green) ![LangGraph](https://img.shields.io/badge/LangGraph-1.0-orange) ![DeepAgents](https://img.shields.io/badge/DeepAgents-0.4-purple) ![Vue3](https://img.shields.io/badge/Vue-3.5-42b883) ![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1) 

</div>

---


## 1. 项目简介

DeepSearch 是一个面向企业场景的**多智能体深度搜索平台**。它以一个「主智能体」（团队负责人）为核心，调度三个各司其职的专家子智能体，打破单一数据源的局限：

| 子智能体 | 数据源 | 定位 |
|---------|--------|------|
| 网络搜索助手 | Tavily API | 获取互联网公开信息 |
| 数据库查询助手 | MySQL | 查询企业内部结构化业务数据（药品/库存/销售等） |
| RAGFlow 助手 | RAGFlow 知识库 | 检索企业私有非结构化文档 |

系统在获取完整信息后，可根据用户指令生成 **Markdown / PDF** 格式的深度研究报告，并通过 WebSocket 将**任务进度、工具调用、子智能体执行过程**实时推送到前端，实现全程可视化。

> 典型场景：分析师向平台提问「分析某药品近半年在华东地区的销售趋势并输出报告」，主智能体自动并行调度「数据库助手」取销售数据、「网络搜索助手」查行业背景、「RAGFlow 助手」查内部制度文档，最后汇总生成一份可下载的 Markdown/PDF 报告。

---

## 2. 核心特性

- 🤖 **多智能体编排**：基于 LangGraph + DeepAgents 构建，主智能体自主决策何时、调用哪个子智能体，支持多轮、递进式信息收集。
- 🔍 **三源异构信息融合**：互联网（Tavily）+ 结构化数据（MySQL）+ 私有知识库（RAGFlow），打破信息孤岛。
- ⚡ **全流程流式执行**：`async/await` 异步流式执行（`main_agent.astream`），支持高并发多会话，互不阻塞。
- 📡 **实时进度可视化**：工具调用、子智能体切换、最终结果均通过 WebSocket 实时推送，前端以「思维过程」折叠面板展示。
- 📄 **多格式文档输出**：Markdown 直接生成，PDF 通过 Word 引擎（COM）转换，支持表格、代码块等复杂排版。
- 📎 **文件上传与语义读取**：支持上传 `.md / .docx / .pdf / .xlsx / .xls` 文件，Agent 可读取并参考内容生成报告。
- 🔒 **会话级隔离与路径安全**：`ContextVar` 保证并发请求数据不串台；统一路径解析 + 下载/列表接口双重越权校验，防止路径穿越攻击。
- 🗂️ **会话文件管理**：每个会话拥有独立工作目录（`output/session_{id}`），前端内嵌文件浏览器，所见即所得。

---

## 3. 系统架构

### 3.1 架构总览

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          Web 前端（Vue 3 + TypeScript + Vite）             │
│             聊天 UI ｜ 文件上传 ｜ 实时进度(思维过程) ｜ 会话文件浏览器        │
└─────────────────────────────────┬─────────────────────────────────────────┘
                                  │ REST API + WebSocket
┌─────────────────────────────────▼─────────────────────────────────────────┐
│                         API 层（FastAPI） api/server.py                    │
│      POST /api/task ｜ POST /api/upload ｜ GET /api/files ｜ GET /api/     │
│      download ｜ WS /ws/{thread_id}（实时双向通道）                          │
└──────────────┬──────────────────────────────────┬─────────────────────────┘
               │ asyncio.create_task               │ 事件推送(ConnectionManager)
┌──────────────▼──────────────────────────────────▼─────────────────────────┐
│              会话上下文隔离层（ContextVar） api/context.py                   │
│        session_dir（工作目录）+ thread_id（会话ID）—— 并发请求数据隔离         │
└──────────────────────────────┬─────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────────────┐
│              智能体编排层（LangGraph + DeepAgents） agent/                  │
│                                                                           │
│                 ┌────────────────────────────────────┐                     │
│                 │   主智能体 main_agent（团队负责人）  │                     │
│                 │  create_deep_agent(model, tools,   │                     │
│                 │      subagents, checkpointer)      │                     │
│                 └──┬──────────────┬──────────────┬───┘                     │
│        ┌───────────▼───┐   ┌──────▼────────┐  ┌──▼────────────┐            │
│        │ 网络搜索助手    │   │ 数据库查询助手  │  │ RAGFlow 助手  │            │
│        │ Tavily 工具    │   │ MySQL 3 工具   │  │ RAGFlow 2 工具│            │
│        └───────┬───────┘   └──────┬────────┘  └──┬────────────┘            │
└───────────────┼──────────────────┼──────────────┼──────────────────────────┘
                │                  │              │
        ┌───────▼──────┐   ┌───────▼────────┐  ┌──▼────────────────┐
        │  Tavily API  │   │  MySQL 8.0     │  │  RAGFlow 服务     │
        │  (互联网)     │   │  (docker)      │  │  (私有知识库)      │
        └──────────────┘   └────────────────┘  └───────────────────┘
```

### 3.2 主智能体工具链

主智能体自身持有三个**文件处理工具**，子智能体专职**信息获取**：

| 层 | 工具 | 来源 |
|----|------|------|
| 主智能体 | `generate_markdown` 生成 Markdown 报告 | `tools/markdown_tools.py` |
| 主智能体 | `convert_md_to_pdf` Markdown 转 PDF（Word 引擎） | `tools/pdf_tools.py` |
| 主智能体 | `read_file_content` 读取上传文件（md/docx/pdf/xlsx/xls） | `tools/upload_file_read_tool.py` |
| 网络搜索助手 | `internet_search` 互联网检索（topic/max_results 可调） | `tools/tavily_tool.py` |
| 数据库查询助手 | `list_sql_tables` / `get_table_data` / `execute_sql_query` | `tools/db_tools.py` |
| RAGFlow 助手 | `get_assistant_list` / `create_ask_delete` | `tools/ragflow_tools.py` |

### 3.3 一次任务的核心流程

```
用户提问
   │
   ▼
① POST /api/task {query, thread_id}          （接口立即返回，任务异步执行）
   │
   ▼
② run_deep_agent(query, session_id)
   ├─ 创建会话工作目录  output/session_{id}
   ├─ 若存在上传文件 → 复制到工作目录 + 注入读取提示词
   ├─ 写入 ContextVar（session_dir / thread_id）  → 会话隔离
   ├─ monitor.report_session_dir  → WebSocket 推送工作目录
   │
   ▼
③ main_agent.astream(...)  主智能体开始流式执行
   ├─ 决策 → 调用子智能体（task 工具）→ monitor.report_assistant → 前端推送
   │        ├─ 网络搜索助手 → Tavily
   │        ├─ 数据库查询助手 → MySQL
   │        └─ RAGFlow 助手 → 知识库
   ├─ 子智能体执行时 → 内部工具埋点 monitor.report_tool → 前端推送
   │
   ▼
④ 主智能体汇总信息 → generate_markdown / convert_md_to_pdf
   │
   ▼
⑤ monitor.report_task_result → 前端渲染最终答案 + 文件卡片
   │
   ▼
⑥ finally: reset 上下文（防止内存泄漏 / 请求串台）
```

---

## 4. 技术栈

| 层次 | 技术选型 | 说明 |
|------|---------|------|
| 后端框架 | **FastAPI 0.129** + Uvicorn | 异步高性能，自动 OpenAPI 文档 |
| 智能体编排 | **LangGraph 1.x** + **DeepAgents 0.4** | 状态图驱动、流式执行、断点续跑 |
| LLM | 通义千问（Qwen）系列，OpenAI 兼容协议 | 默认 `qwen-max`，可切换 `qwen3-32b` 等 |
| 网络搜索 | **Tavily** | 面向 AI 的搜索 API，支持 topic 分类 |
| 数据库 | **MySQL 8.0**（官方驱动） | 企业业务数据（商品/库存/销售） |
| 知识库 | **RAGFlow SDK** | 私有文档 RAG 检索 |
| 文档生成 | python-markdown + **Microsoft Word COM** | PDF 经 Word 引擎转换（Windows 专属） |
| 实时通信 | **WebSocket**（FastAPI） | 进度与结果实时推送 |
| 并发隔离 | **ContextVar** | 协程级变量，避免异步请求串台 |
| 前端 | **Vue 3 + TypeScript + Vite**（rolldown-vite） | marked 渲染 Markdown、axios 通信 |
| 容器化 | Docker / docker-compose | MySQL 一键部署 |

---

## 5. 快速开始

### 5.1 环境要求

| 依赖 | 版本要求 | 说明 |
|------|---------|------|
| Python | ≥ 3.10（推荐 3.13） | 项目使用 `async` / `ContextVar` / `Path.is_relative_to` 等特性 |
| Node.js | ≥ 18 | 前端构建 |
| MySQL | 8.0 | 可通过 Docker 一键启动 |
| Windows + MS Word | PDF 转换必需 | `pdf_tools` 依赖 `pywin32` COM 接口，**Linux/macOS 不支持 PDF 转换** |
| 外部服务 | — | Tavily API Key、DashScope（通义千问）API Key、RAGFlow 服务 |

### 5.2 安装步骤

```bash
# ① 克隆项目
git clone <your-repo-url> deep_search
cd deep_search

# ② 创建 Python 虚拟环境并安装依赖
python -m venv venv
# Windows
venv\Scripts\activate
# Linux / macOS
# source venv/bin/activate

pip install -r requirements.txt

# ③ 配置环境变量
cp .env.example .env
# 编辑 .env，填入真实的 API Key（详见 §10 配置说明）
```

### 5.3 启动 MySQL（可选，若已有数据库可跳过）

```bash
docker-compose up -d mysql
```

> 默认创建数据库 `pharma_db`（root / root），若为全新部署，请按业务建表后使用。

### 5.4 启动后端

```bash
uvicorn api.server:app --host 0.0.0.0 --port 8000 --reload
# 或
python -m api.server
```

启动成功后：
- API 文档（Swagger UI）：http://localhost:8000/docs
- 首次导入 `agent.main_agent` 会初始化 LLM 与智能体图，耗时数秒属正常现象

### 5.5 启动前端

```bash
cd ui
npm install
npm run dev
```

浏览器访问 Vite 输出的地址（默认 http://localhost:5173）。

> ⚠️ 前端当前硬编码后端地址为 `http://localhost:8000`（见 `ui/src/App.vue`），请确保后端端口为 8000；跨域已由后端 CORS 中间件放开。

### 5.6 验证

在聊天框输入例如：

> 「查询数据库中销量排名前 5 的药品，结合网络资料分析其市场前景，生成一份 Markdown 报告」

预期看到：右侧「思维过程」实时展示工具调用与子智能体切换 → 最终答案 → 文件卡片可点击下载。

---

## 6. 使用指南

### 6.1 支持的问题类型

| 类型 | 示例 | 系统行为 |
|------|------|---------|
| 纯信息查询 | 「某药品的规格和库存是多少？」 | 子智能体检索 → 直接文本回答，不生成文件 |
| 深度研究 + 文档生成 | 「分析华东区销售趋势并生成报告」 | 多源检索 → 生成 Markdown / PDF 文件 |
| 基于上传文件 | 上传 Excel 后问「统计这批数据」 | 读取上传文件 → 结合外部信息回答 |

### 6.2 文件上传

1. 点击输入框左侧的 📎 按钮，可多选文件；
2. 支持格式：`.md`、`.txt`、`.docx`、`.pdf`、`.xlsx`、`.xls`；
3. 上传后文件会保存至 `updated/session_{thread_id}`，Agent 收到明确指令优先读取这些文件；
4. 发送消息时随请求一并上传。

### 6.3 文件管理

- 每个会话的文件存储在 `output/session_{thread_id}` 目录；
- 前端右侧「Session Files」面板实时列出生成文件，支持在线预览与下载；
- 接口 `GET /api/files` 与 `GET /api/download` 已做越权校验，仅允许访问 `output/` 目录。

### 6.4 会话机制

- 前端使用 `crypto.randomUUID()` 生成持久会话 ID（页面刷新不丢失）；
- 同一 `thread_id` 的多轮提问会共享上下文（LangGraph `InMemorySaver` 检查点，基于内存，**服务重启后历史丢失**）；
- 每个会话拥有独立工作目录，多会话并发互不干扰。

---

## 7. 项目结构

```
deep_search/
├── agent/                          # 智能体层（核心编排逻辑）
│   ├── llm.py                      # LLM 初始化（OpenAI 兼容协议 → 通义千问）
│   ├── main_agent.py               # 主智能体：创建 + 异步流式执行 run_deep_agent
│   ├── prompts.py                  # 加载 prompt/prompts.yml
│   └── subagents/                  # 三个专家子智能体
│       ├── network_search_agent.py # 网络搜索助手（Tavily）
│       ├── database_query_agent.py # 数据库查询助手（MySQL）
│       └── knowledge_base_agent.py # RAGFlow 知识库助手
├── api/                            # API 层（FastAPI）
│   ├── server.py                   # REST + WebSocket 入口、上传/下载/文件列表
│   ├── monitor.py                  # ToolMonitor 埋点单例 + ConnectionManager
│   └── context.py                  # ContextVar 会话上下文（隔离并发请求）
├── prompt/
│   └── prompts.yml                 # 主/子智能体的角色提示词（可配置中心）
├── rawflow/                        # RAGFlow 独立脚本（演示/运维）
│   ├── rag_config.py               # RAGFlow 环境变量加载
│   ├── chat_assistant_demo.py      # 助手列表查询 / 提问演示
│   └── knowledge_demo.py           # 知识库创建 / 文件上传脚本
├── tools/                          # 工具层（LangChain @tool）
│   ├── tavily_tool.py              # internet_search 网络搜索
│   ├── db_tools.py                 # MySQL 查询三件套
│   ├── ragflow_tools.py            # RAGFlow 助手列表 / 提问
│   ├── markdown_tools.py           # generate_markdown
│   ├── pdf_tools.py                # convert_md_to_pdf（Word 引擎）
│   └── upload_file_read_tool.py    # read_file_content（多格式读取）
├── ui/                             # 前端（Vue 3 + TS + Vite）
│   └── src/App.vue                 # 聊天界面 + 文件上传 + 实时进度 + 文件浏览器
├── utils/
│   ├── path_utils.py               # resolve_path 统一路径解析（防穿越/防嵌套）
│   └── word_converter.py           # MD → HTML → Word COM → PDF
├── output/                         # 运行时生成（gitignore）：会话工作目录
├── updated/                        # 运行时生成（gitignore）：用户上传文件
├── .env.example                    # 环境变量模板
├── docker-compose.yml              # MySQL 8.0 一键启动
├── requirements.txt                # Python 依赖（版本锁定）
└── README.md                       # 本文档
```

---

## 8. API 参考

Base URL：`http://localhost:8000` ｜ 交互式文档：`/docs`

### 8.1 POST `/api/task` — 提交任务

创建深度搜索任务，接口**立即返回**，任务异步执行，结果通过 WebSocket 推送。

```json
// Request
{
  "query": "分析某药品销售趋势并生成报告",
  "thread_id": "可选，不传则服务端生成 UUID"
}

// Response 200
{ "status": "started", "thread_id": "xxx" }
```

### 8.2 POST `/api/upload` — 上传文件

`multipart/form-data`：字段 `files`（可多个）+ `thread_id`（必填）。

```json
// Response 200
{ "status": "uploaded", "files": ["a.pdf", "b.xlsx"] }
```

### 8.3 GET `/api/files` — 文件列表

| 参数 | 说明 |
|------|------|
| `path` | 目标目录绝对路径（必须在 `output/` 下，否则拒绝访问） |

返回按修改时间倒序的文件数组：`name / type / path / size / mtime`。

### 8.4 GET `/api/download` — 文件下载

| 参数 | 说明 |
|------|------|
| `path` | 文件绝对路径（必须在 `output/` 下） |

返回 `FileResponse` 文件流，浏览器自动下载。

> 安全提示：`/api/files` 与 `/api/download` 均使用 `Path.resolve()` + `is_relative_to()` 双重校验，拒绝路径越权与目录遍历攻击。

### 8.5 WebSocket `/ws/{thread_id}`

建立会话级实时通道（详见 [§9](#9-websocket-实时协议)）。

---

## 9. WebSocket 实时协议

客户端连接：`ws://localhost:8000/ws/{thread_id}`，断线后前端每 3 秒自动重连。

**服务端 → 客户端**消息统一结构：

```json
{
  "type": "monitor_event",
  "event": "<事件类型>",
  "message": "人类可读描述",
  "data": {},
  "timestamp": "2026-08-01T15:00:00.000000"
}
```

### 事件类型一览

| event | 触发时机 | data 关键字段 |
|-------|---------|--------------|
| `session_created` | 会话工作目录创建 | `path`（工作目录绝对路径） |
| `tool_start` | 任意工具开始执行 | `tool_name`、`args` |
| `assistant_call` | 主智能体调用子智能体 | `assistant_name`、`args`（含 description） |
| `task_result` | 任务最终结果产出 | `result`（最终回答文本） |
| `error` | 执行异常 | —（message 携带错误信息） |

**客户端 → 服务端**：发送任意文本（通常为 `ping` 心跳），服务端回复：

```json
{ "type": "pong", "message": "服务端已收到: ..." }
```

> 前端对 `tool_start` 事件中的 `args.filename` 自动生成文件卡片（拼接会话 URL 实现下载链接）。

---

## 10. 配置说明

所有配置通过项目根目录 `.env` 注入（参考 `.env.example`），修改后需重启后端。

| 变量 | 必填 | 默认值 | 说明 |
|------|------|--------|------|
| `RAGFLOW_API_URL` | ✅ | — | RAGFlow 服务地址，如 `http://your-ragflow-server:port` |
| `RAGFLOW_API_KEY` | ✅ | — | RAGFlow API Key（格式 `ragflow-...`） |
| `OPENAI_BASE_URL` | ✅ | DashScope 兼容地址 | LLM 服务地址，默认 `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| `OPENAI_API_KEY` | ✅ | — | DashScope 通义千问 API Key |
| `LLM_QWEN2.5` | 否 | `qwen2.5-14b-instruct` | 候选模型（备用） |
| `LLM_QWEN3` | 否 | `qwen3-32b` | 候选模型（备用） |
| `LLM_QWEN_MAX` | ✅ | `qwen-max` | **实际使用的模型**（`agent/llm.py` 读取） |
| `TAVILY_API_KEY` | ✅ | — | Tavily 搜索 API Key（格式 `tvly-...`） |
| `MYSQL_USER` | ✅ | `root` | 数据库用户 |
| `MYSQL_PASSWORD` | ✅ | `root` | 数据库密码 |
| `MYSQL_DATABASE` | ✅ | `pharma_db` | 数据库名 |
| `MYSQL_HOST` | 否 | `localhost` | 数据库地址 |
| `MYSQL_PORT` | 否 | `3306` | 数据库端口 |

> 🔑 获取方式：DashScope Key 在阿里云百炼控制台创建；Tavily Key 在 tavily.com 注册。

---

## 11. 子智能体说明

所有角色提示词集中在 **`prompt/prompts.yml`**，为「提示词即配置」设计，修改无需改代码。

### 11.1 网络搜索助手（Tavily）

- **职责**：检索互联网公开信息（背景知识、行业动态、外部数据）。
- **工具**：`internet_search(query, topic, max_results, include_raw_content)`。
- **行为约束**：至少从 3 个角度检索，最多 5 次；不可用于数据库 / 知识库数据。

### 11.2 数据库查询助手（MySQL）

- **职责**：查询企业结构化业务数据（药品信息、库存、销售记录等）。
- **工具**：`list_sql_tables`（列表）→ `get_table_data`（预览前 100 行）→ `execute_sql_query`（自定义 SQL）。
- **行为约束**：强制先列表再查询；不包含概括性知识，只含具体商品数据。

### 11.3 RAGFlow 助手（知识库）

- **职责**：检索企业私有非结构化文档。
- **工具**：`get_assistant_list`（枚举助手与其关联知识库）→ `create_ask_delete`（临时会话提问后即删）。
- **行为约束**：必须依据助手描述提问；至少提问 3 个不同角度的问题；保留原始信息不概括。

### 11.4 主智能体（团队负责人）

- **编排规则**：先信息获取、后文件生成（**禁止**先调用生成工具）；生成的 todo-list 规划任务；生成文件 ≥1000 字。
- **文件职责**：所有文档生成由主智能体独立完成，不把路径告知子智能体。

---

## 12. 部署与运维

### 12.1 Docker Compose（数据库）

```bash
docker-compose up -d mysql     # 启动 MySQL 8.0（utf8mb4，数据卷持久化）
docker-compose down            # 停止（数据保留在 mysql_data 卷）
```

### 12.2 生产部署建议

| 关注点 | 建议 |
|--------|------|
| ASGI 服务 | `uvicorn api.server:app --host 0.0.0.0 --port 8000 --workers 2`（注意多 worker 下 InMemorySaver 不共享，会话记忆按 worker 分片） |
| 反向代理 | Nginx 转发 `/api` 与 `/ws`（WebSocket 需配置 Upgrade 头），并挂载 `output/` 静态目录 |
| 会话记忆 | `InMemorySaver` 仅内存态，重启即失；生产可替换为 LangGraph 持久化后端（如 Redis/Postgres checkpointer） |
| 前端部署 | `cd ui && npm run build` → 产物在 `ui/dist`，由 Nginx 托管；需将 `App.vue` 中硬编码的 API 地址改为网关地址 |
| 环境隔离 | 开发/测试/生产使用独立 `.env` 与数据库实例 |
| 日志监控 | `output/` 与 `updated/` 目录纳入磁盘监控；为 8000 端口配置健康检查（`/docs` 或自定义探针） |
| 密钥管理 | API Key 建议接入密钥管理系统（如 Vault / 云 KMS），避免明文落盘 |

### 12.3 数据目录说明

| 目录 | 用途 | 生命周期 |
|------|------|---------|
| `output/session_{id}` | 会话生成文件 | 建议按会话保留策略定时归档/清理 |
| `updated/session_{id}` | 用户上传文件 | 同上 |

---

## 13. 安全设计

本项目内置了以下安全措施，生产使用仍需遵循最小权限原则：

1. **路径越权防护**：`/api/files`、`/api/download` 通过 `Path.resolve()` + `is_relative_to()` 校验，强制限定在 `output/` 目录，杜绝目录遍历。
2. **会话数据隔离**：ContextVar（`session_dir` / `thread_id`）实现协程级隔离，并发请求不串台；`finally` 中 reset，防内存泄漏。
3. **路径解析防嵌套**：`resolve_path` 清洗虚拟路径前缀（`/workspace`、`/mnt/data`、`/home/user`），检测 `session_{id}/session_{id}` 嵌套并修正。
4. **SQL 安全提示**：`execute_sql_query` 接受 LLM 生成的自定义 SQL，属**高权限动态查询**——生产环境建议使用只读账号，并限制 DDL/DML 权限（仅授权 SELECT）。
5. **YAML 安全加载**：`prompts.py` 使用 `yaml.safe_load`，避免反序列化注入。
6. **密钥保护**：`.env` 已加入 `.gitignore`，严禁提交仓库。
7. **CORS**：当前为 `allow_origins=["*"]`，生产环境应收紧为具体可信域名。

> ⚠️ **已知风险**：`/api/download` 与 `/api/files` 当前无认证鉴权。企业内部使用时请务必在前置网关（Nginx / API 网关）增加身份认证与访问控制。

---

## 14. 已知限制与 FAQ

### 已知限制

| 限制 | 说明 |
|------|------|
| PDF 转换依赖 Windows | `word_converter.py` 使用 `win32com` + 本机 Microsoft Word，非 Windows 或未装 Word 则 PDF 生成失败 |
| 会话记忆为内存态 | `InMemorySaver` 重启即丢，多 worker 下记忆不共享 |
| 前端地址硬编码 | `ui/src/App.vue` 中 API/WS 地址写死为 `localhost:8000`，跨环境部署需改造为配置化 |
| 依赖体积较大 | `requirements.txt` 含 135 个锁定包（含大量文档处理库），首次安装耗时较长 |
| SQL 注入面 | 动态 SQL 由 LLM 生成，权限需严格收敛 |

### FAQ

**Q1：启动报错 `Missing OpenAI API key`？**
检查 `.env` 是否已创建且 `OPENAI_API_KEY` / `OPENAI_BASE_URL` 填写正确；确认 `agent/llm.py` 读取的 `LLM_QWEN_MAX` 非空。

**Q2：提示「缺少依赖库 pywin32」？**
PDF 转换需在 Windows 环境安装 `pywin32` 并安装 Microsoft Word；若无需 PDF 功能，可忽略该错误（Markdown 生成不受影响）。

**Q3：数据库查询报错「缺失数据库核心配置」？**
确认 `.env` 中 `MYSQL_USER`、`MYSQL_PASSWORD`、`MYSQL_DATABASE` 三项均存在且非空；`docker-compose up -d mysql` 启动后等待 10-20 秒初始化。

**Q4：RAGFlow 工具返回「无可用助手」？**
确认 `RAGFLOW_API_URL` / `RAGFLOW_API_KEY` 正确、RAGFlow 服务可达，且已在 RAGFlow 后台创建聊天助手并关联知识库。[RAGFlow](https://ragflow.io/docs) 部署使用可参考官方文档。

**Q5：前端页面显示错误 / WebSocket 重连失败？**
确认后端运行在 8000 端口；浏览器控制台检查跨域（CORS）与 WebSocket 连接日志。

**Q6：多用户并发会不会串数据？**
不会。系统基于 `ContextVar` + 会话目录隔离，每个请求链路持有独立的 `session_dir` 与 `thread_id`（见 `api/context.py` 设计注释）。

---

## 15. 开发指南

### 15.1 新增一个子智能体

```python
# ① 在 prompt/prompts.yml 的 sub_agents 下新增角色配置（name/description/system_prompt）
# ② 编写工具（如 tools/my_tool.py），使用 @tool 装饰并埋点：
from langchain_core.tools import tool
from api.monitor import monitor

@tool
def my_tool(query: str) -> str:
    """工具描述（LLM 依据此决定是否调用）"""
    monitor.report_tool("我的新工具", {"query": query})
    ...
    return result

# ③ 在 agent/subagents/ 下创建子智能体定义
from agent.prompts import sub_agents_content
new_agent = {
    "name": sub_agents_content['xxx']['name'],
    "description": sub_agents_content['xxx']['description'],
    "system_prompt": sub_agents_content['xxx']['system_prompt'],
    "tools": [my_tool],
}

# ④ 在 agent/main_agent.py 的 subagents 列表注册
```

### 15.2 新增主智能体工具

在 `tools/` 下新建 `@tool` 函数（遵守类型注解 + docstring + 埋点三要素），导入后加入 `create_deep_agent(tools=[...])` 即可。

### 15.3 编码规范（约定）

- 工具函数一律使用 `@tool` 装饰器并携带**参数类型注解**与中文 docstring；
- 每个工具入口调用 `monitor.report_tool(tool_name, args)` 完成埋点；
- 文件读写一律经过 `resolve_path(filename, session_dir)` 统一路径解析；
- 资源使用 `with` 上下文管理（数据库连接、文件句柄）；
- 异常需捕获并返回可读信息，避免裸抛导致任务中断。

### 15.4 本地调试

```bash
# 单测某个工具（工具文件自带 __main__ 测试入口）
python tools/markdown_tools.py
python tools/db_tools.py
```

---

## 16. 贡献指南

1. Fork 仓库并创建特性分支：`git checkout -b feature/xxx`；
2. 遵循 [§15 开发指南](#15-开发指南) 的编码规范；
3. 提交信息使用清晰的中文或英文描述，如 `feat: 新增 XX 子智能体`；
4. 发起 Pull Request，附上改动说明与测试结果。

---

## License

[MIT](https://opensource.org/licenses/MIT)