# LASPAI 智能体系统总体设计方案

本文档描述 LASPAI 智能体项目的整体架构设计，涵盖智能体端、MCP 服务端和数据库三大部分的分工与协作。

---

## 一、项目定位

LASPAI 智能体是一个面向材料科学的 AI 智能体系统，核心能力包括：

- **计算建模**：通过自然语言驱动分子生成、晶体结构搜索、表面建模、吸附构型优化、分子动力学、全局优化、性质计算等化学计算任务
- **RAG 问答**：基于知识库文档的智能问答

系统采用**三层解耦架构**，将「智能编排」「计算执行」「数据存储」三者彻底分离。

---

## 二、总体架构

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                              前端 (Vue)                                  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │ SSE / REST
                               ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    智能体端 (lasp_agent)                                  │
│                                                                          │
│  职责：理解用户意图、编排工具调用、生成回复、SSE 推送、直连数据库读写         │
│                                                                          │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │  api/    │  │  main_agent/ │  │  sub_agents/ │  │      ui/        │ │
│  │ HTTP 路由 │  │  主智能体编排  │  │  子智能体群    │  │   SSE 渲染      │ │
│  └──────────┘  └──────────────┘  └──────┬───────┘  └─────────────────┘ │
│                                         │                                │
│                  ┌──────────────────────┼──────────────────────┐        │
│                  │      services/       │     MCP Client       │        │
│                  │  系统编排 & 会话管理   │   (仅计算工具调用)     │        │
│                  └──────────────────────┼──────────────────────┘        │
│                                         │                                │
│                  ┌──────────────────────┼──────────────────────┐        │
│                  │        core/         │   直连数据库           │        │
│                  │  实体 / 配置 / 日志   │  (数据存取，不走 MCP)   │        │
│                  └──────────────────────┼──────────────────────┘        │
└───────────────────────────┬─────────────┼───────────────────────────────┘
                            │             │
              直连 DB       │             │ MCP 协议
             (数据存取)      │             │ (仅计算工具)
                            ▼             ▼
┌───────────────────────────┐  ┌──────────────────────────────────────────┐
│   数据库 (唯一事实来源)      │  │      MCP 服务端 (lasp_mcp_server)        │
│                           │  │                                          │
│  ┌─────────────────────┐  │  │  职责：暴露标准化计算工具、调用计算集群、     │
│  │  化学数据（结构文件、  │◀─┼──│        结果入库                            │
│  │  计算结果、元数据）    │  │  │                                          │
│  ├─────────────────────┤  │  │  对外提供：REST 文件上传 + MCP 资源下载     │
│  │  会话/消息记录        │  │  │                                          │
│  ├─────────────────────┤  │  │  接收 tool call → 提取数据 → 调用计算集群   │
│  │  用户 LLM 配置       │  │  │  → 结果入库 → 返回短 ID                    │
│  └─────────────────────┘  │  │                                          │
│                           │  │  ┌──────────┐ ┌────────────┐             │
│  智能体端和 MCP Server    │  │  │  api/    │ │mcp_handlers│             │
│  共享同一数据库实例        │  │  │ REST路由  │ │  MCP 工具   │             │
└───────────────────────────┘  │  ├──────────┤ ├────────────┤             │
                               │  │ services/│ │   core/    │             │
                               │  │ 业务逻辑  │ │ 基础设施    │             │
                               │  └──────────┘ └────────────┘             │
                               └──────────────────────────────────────────┘
```

---

## 三、三部分职责

### 3.1 智能体端（`lasp_agent/`）

| 维度           | 说明                                                          |
| -------------- | ------------------------------------------------------------- |
| **核心职责**   | 理解用户意图、编排工具调用、生成回复、SSE 推送                |
| **数据存取**   | **直连数据库**读写所有数据（内部通路，不走 MCP）              |
| **计算调用**   | 通过 `MCPClient` → MCP Server 调用计算工具                    |
| **状态持久化** | 自定义 LangGraph checkpointer（覆盖模式）+ 业务数据库定时快照 |
| **日志追踪**   | OpenTelemetry + W3C `traceparent` 全链路                      | POST /chat 为 Root Span，自动插桩（FastAPI/LangChain/SQLAlchemy），跨服务通过 `traceparent` Header 继承 trace_id |
| **技术栈**     | FastAPI + LangGraph + MCP SDK (client) + SQLModel             |

> 详细设计见 → [LASPAI 智能体端设计方案](./LASPAI%20智能体端设计方案.md)

### 3.2 MCP 服务端（`lasp_mcp_server/`）

| 维度           | 说明                                                                 |
| -------------- | -------------------------------------------------------------------- |
| **核心职责**   | 暴露标准化计算工具、调用计算集群、结果入库                           |
| **对智能体端** | 提供 MCP 工具列表供 LLM function calling 使用                        |
| **对外部客户** | REST 文件上传 + MCP 资源下载（完备的独立产品接口）                   |
| **鉴权**       | SSE 连接时校验 Token，`user_id` 注入 `contextvars`，实现底层权限隔离 |
| **短 ID 生成** | 带语义前缀的短 ID（如 `mol_8f3a9b`），防 LLM 幻觉                    |
| **技术栈**     | FastAPI + MCP SDK (server) + SQLModel                                |

> 详细设计见 → [LASPAI MCP Server 重构方案](./LASPAI%20MCP%20Server%20重构方案.md)

### 3.3 数据库

> **⚠️ TODO（他人负责）**：此部分由他人负责，此处仅描述接口约束。以下内容需由数据库负责人补充完整。

| 维度         | 说明                                                                                     |
| ------------ | ---------------------------------------------------------------------------------------- |
| **定位**     | 唯一事实来源，智能体端和 MCP Server 共享同一数据库实例                                   |
| **存储内容** | 化学结构文件（PDB/CIF）、计算结果、元数据、短 ID 映射、会话记录、消息记录、用户 LLM 配置 |
| **访问方式** | 智能体端直接 SQL 查询；MCP Server 通过自身 data access 层查询                            |
| **TODO**     | 具体数据库选型（PostgreSQL / MySQL / SQLite）待定                                        |
| **TODO**     | 表结构详细设计（字段、索引、外键关系）待补充                                             |
| **TODO**     | 数据迁移策略与版本管理方案待补充                                                         |
| **TODO**     | 备份恢复方案待补充                                                                       |

**关键约束**：

- 所有化学数据必须通过短 ID（`{type}_{random}`）索引，不允许直接暴露数据库主键
- 数据库表结构需同时支持智能体端直连查询和 MCP Server 的 DAO 层访问
- `user_id` 作为数据隔离的强制字段，所有查询必须带用户过滤条件

---

## 四、数据流

### 4.1 计算任务流程（用户 → 结果）

```text
1. 用户上传结构文件
   前端 → POST /api/agent/chat/{session_id}
   └→ 智能体端 api/chat.py：文件内容直接写入数据库，获得短 ID "crys_abc123"

2. 主智能体编排
   主智能体 LLM 分析用户意图，决定调用 crys_agent
   └→ router_node 构造 SubAgentInput(resource_ids=["crys_abc123"])

3. 子智能体执行
   crys_agent init_node：直连数据库读取 "crys_abc123" 的 CIF 内容
   crys_agent planner_node：LLM 制定执行计划
   crys_agent gen_opt_node：
     └→ mcp_client.call_tool("crystal_gen_opt", {"input_id": "crys_abc123"})
        └→ MCP Server：从数据库提取结构 → 调用计算集群 → 结果入库
        └→ 返回 {"status": "success", "artifact_id": "crys_def456", "energy": -100.5, ...}

4. 结果回传
   crys_agent pack_node：LLM 生成描述，构造 Artifact(id="crys_def456", ...)
   └→ post_process_node：注册到 inventory，ToolMessage 回执
   └→ 回到 LLM_node，LLM 可继续操作或回复用户
```

### 4.2 数据访问路径

```text
                    ┌─ 文件上传 ──────────────┐
                    │  智能体端直接写库          │
用户 ────▶ 智能体端 ──┤                          ├──▶ 数据库
                    │  读取结构 ──────────────│
                    │  智能体端直接查库          │
                    └─────────────────────────┘

                    ┌─ 计算工具调用 ────────────┐
                    │  MCP 协议                  │
智能体端 ──MCP──▶ MCP Server ──┤                          ├──▶ 数据库
                    │  结果入库                   │         (同一实例)
                    └────────────────────────────┘

                    ┌─ 文件上传（外部客户）────────┐
第三方 ──REST──▶ MCP Server ──┤                          ├──▶ 数据库
                    │  文件下载（外部客户）────────┤
第三方 ──MCP──▶ MCP Server ──┘
```

---

## 五、目录结构全景

```text
lasp_agent/                        # 智能体端
├── server.py
├── api/                           # HTTP 接口层
│   ├── deps.py
│   ├── schemas.py
│   └── routers/
│       ├── chat.py
│       ├── sessions.py
│       ├── llm_config.py
│       └── parse.py
├── core/                          # 纯基础设施
│   ├── config/
│   │   ├── env_settings.py
│   │   ├── models.py
│   │   ├── paths.py
│   │   └── setup_logging.py
│   ├── entities/                  # 化学数据实体
│   │   ├── base.py
│   │   ├── molecule.py
│   │   ├── crystal.py
│   │   ├── phonon.py
│   │   ├── spectral.py
│   │   ├── md.py
│   │   ├── go.py
│   │   ├── collection.py
│   │   └── helpers.py
│   ├── database.py                # 数据库引擎（直连）
│   ├── checkpointer.py            # 自定义 checkpointer（覆盖模式）
│   ├── telemetry.py              # OpenTelemetry 初始化：TracerProvider、自动插桩注册
│   ├── context.py                 # contextvars：user_id、user_token
│   └── mcp_client.py              # MCP 客户端（仅计算）
├── services/                      # 应用编排
│   ├── system.py
│   └── session.py
├── main_agent/                    # 主智能体
│   ├── graph.py
│   ├── state.py
│   ├── prompts.py
│   ├── constants.py
│   ├── utils.py
│   ├── llm_presenter.py
│   └── tools/
│       ├── base.py
│       ├── manager.py
│       └── reader/
├── sub_agents/                    # 子智能体群
│   ├── manager.py
│   ├── base/
│   │   ├── blueprint.py
│   │   ├── nodes.py
│   │   └── state.py
│   ├── mol_agent/
│   ├── crys_agent/
│   ├── surf_agent/
│   ├── adsorp_agent/
│   ├── prop_agent/
│   ├── md_agent/
│   ├── go_agent/
│   └── db_query_agent/
├── ui/                            # 前端交互适配
│   ├── manager.py
│   ├── sse_adapter.py
│   └── base/
│       ├── handler.py
│       ├── state.py
│       ├── element.py
│       └── artifact_helper.py
├── ask_service/                   # RAG 问答
│   ├── data/
│   ├── doc_loader.py
│   ├── graph.py
│   └── prompts.py
├── utils/
│   └── thermo_formatter.py
├── requirements.txt
└── .env

lasp_mcp_server/                   # MCP 服务端
├── core/
│   ├── config.py
│   ├── database.py
│   ├── security.py
│   ├── middleware.py
│   └── logger.py
├── api/
│   ├── mcp_routes.py
│   └── upload_routes.py
├── mcp_handlers/
│   ├── server.py
│   ├── resources.py
│   └── tools/
│       ├── molecule_tools.py
│       ├── crystal_tools.py
│       ├── surface_tools.py
│       └── adsorp_tools.py
├── services/
│   └── chemistry_client.py
├── logs/
├── main.py
├── requirements.txt
└── .env

database/                          # 数据库 —— ⚠️ TODO（他人负责）
├── schema/                        # 表结构定义 —— ⚠️ TODO
│   ├── artifact.sql               # ⚠️ TODO 化学数据表
│   ├── session.sql                # ⚠️ TODO 会话表
│   └── ...                        # ⚠️ TODO 完整表结构待补充
└── migrations/                    # ⚠️ TODO 迁移脚本
```

---

## 六、关键技术决策

| 决策             | 选择                              | 说明                                                                                    |
| ---------------- | --------------------------------- | --------------------------------------------------------------------------------------- |
| 三层拆分         | 智能体端 / MCP 服务端 / 数据库    | 编排、计算、存储彻底解耦                                                                |
| 智能体端数据存取 | **直连数据库**                    | 内部使用时不绕 MCP，减少一跳网络开销                                                    |
| MCP 协议边界     | **仅计算工具调用**                | MCP 只传计算指令和返回短 ID，不传文件内容                                               |
| 对外数据存取     | MCP Server REST + resource        | MCP 原生缺上传接口，由 Server 侧补全，对外产品功能完备                                  |
| 子智能体图结构   | 各自独立                          | mol_agent 画板中断等个性化流程不可强行统一                                              |
| 节点执行模型     | 全部 async                        | MCP 调用为异步 I/O                                                                      |
| Checkpointer     | 自定义覆盖模式                    | 替代 LangGraph 原生追加模式                                                             |
| 短 ID 机制       | `{type}_{random}`                 | 由 MCP Server 生成，防 LLM 幻觉，节省 Token                                             |
| 全链路追踪       | OpenTelemetry + W3C `traceparent` | 自动插桩无侵入；跨服务标准透传；暂不部署 APM；Span Attributes 绑定 user_id + session_id |
| 问答知识库       | `ask_service/data/`               | 物理隔离计算数据                                                                        |
| 数据库 schema    | 他人负责                          | 智能体端和 MCP Server 共享同一数据库                                                    |

---

## 七、协作协议

### 7.1 智能体端 ↔ MCP 服务端

```text
智能体端                                          MCP 服务端
───────                                          ──────────
MCPClient.list_tools()         ───────────────▶ 返回工具列表（名称 + JSON Schema）
MCPClient.call_tool(name, args) ───────────────▶ 执行计算 → 返回 {status, artifact_id, metadata}
                      Header: traceparent: 00-{trace_id}-{span_id}-01  → 继承 trace context
```

### 7.2 智能体端 ↔ 数据库

```text
智能体端                                         数据库
───────                                         ──────
core/database.py              ───────────────▶  直连 SQL 查询
  - 读写 Artifact 元数据                         （与 MCP Server 共享同一实例）
  - 读写会话/消息记录
  - 读写用户 LLM 配置
```

### 7.3 MCP 服务端 ↔ 外部客户（售卖场景）

```text
外部客户                                         MCP 服务端
───────                                         ──────────
POST /api/v1/files/upload     ───────────────▶ 文件存入数据库 → 返回 artifact_id
GET  artifact://{type}/{id}   ───────────────▶ 从数据库读取 → 返回文件内容
MCPClient.call_tool(...)      ───────────────▶ 执行计算（同上）
```

---

> **关联文档**：
> - [LASPAI 智能体端设计方案](./LASPAI%20智能体端设计方案.md)
> - [LASPAI MCP Server 设计方案](./LASPAI%20MCP%20Server%20设计方案.md)
