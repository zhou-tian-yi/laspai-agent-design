# LASPAI MCP Server 设计方案

本文档概述了将计算化学智能体底层工具调用迁移至 MCP (Model Context Protocol) 架构的服务器端功能需求与目录结构设计。

---

## 一、 核心功能架构

### 1. 基础设施与传输层 (Infrastructure & Transport)
* **FastAPI 融合**：采用 FastAPI 作为底层应用框架，处理全局中间件、CORS 以及服务生命周期管理。
* **全链路追踪**：智能接力上游 `traceparent` Header——智能体端发来请求时继承其 `trace_id` 创建子 Span；第三方 REST API 直接调用时由 OTel 自动生成全新 Root Span。Span Attributes 绑定 `user_id`（从 Token 解析）、`tool_name`、`job_id`（计算任务维度）。
* **统一日志管理 (Logging)**：配置 OTel 将 trace/span 上下文桥接到 Python `logging` 模块，通过自定义 Log Formatter 将 `trace_id`、`span_id`、`user_id` 自动注入到每行日志中，持久化记录 API 调用、鉴权状态、工具执行耗时及内部异常。
* **SSE 路由接管**：使用 MCP SDK 的 `SseServerTransport` 接管 `/mcp/sse`（用于客户端建立长连接）和 `/mcp/messages`（用于接收客户端的 JSON-RPC 消息）。
* **无状态上下文鉴权**：采用无状态 Token 校验机制。要求客户端在 `GET /mcp/sse` 和 `POST /mcp/messages` 的 HTTP Header 中均携带鉴权 Token。
* **ASGI 级路由拦截**：针对使用底层 `Mount` 挂载的 `/mcp/messages` 路由，编写自定义 ASGI 包装器（Wrapper）进行拦截。在请求进入 MCP SDK 处理之前，提取并校验 Token，将 `user_id` 实时注入当前独立请求的 `contextvars` 异步上下文中，供后方的 `@mcp.tool` 和 `@mcp.resource` 安全读取，实现多租户权限隔离。

### 2. 数据持久化层 (Database & Storage)
* **结构入库与权限绑定**：设计统一的数据库表（如 `ArtifactModel`），将化学结构文件（PDB/CIF 文本）、元数据以及所属的 `user_id` 统一持久化存储。
* **短 ID 生成机制**：舍弃长 UUID，生成安全的、带语义前缀的短 ID（如 `mol_8f3a9b`、`crys_2b19c`）作为大模型在工具链中传递的主键参数，防范大模型幻觉与 Token 浪费。

### 3. MCP 核心能力 (Capabilities)
* **计算工具 (`@mcp.tool`)**：
  * 接收大模型传来的短 ID（如 `input_id="mol_abc123"`）。
  * 结合上下文中的 `user_id`，从数据库安全提取结构文本。
  * 调用底层的化学计算集群 API（生成、优化、振动等）。
  * 计算完成后，将新生成的结构存入数据库，仅向大模型返回结构化的元数据和新的短 ID（如 `{"status": "success", "energy": -100.5, "id": "mol_def456"}`）。
* **资源读取 (`@mcp.resource`)**：
  * 注册资源 URI 模板（如 `artifact://{resource_type}/{primary_key}`）。
  * 当智能体前端或外部应用请求此 URI 时，服务器通过提取出的主键和上下文 `user_id` 查询数据库，鉴权通过后直接返回结构的纯文本内容。

### 4. 混合 API (Custom Routes)
* **独立文件上传接口**：提供标准的 `POST /api/v1/files/upload` 接口，接收前端用户手动上传的结构文件，验证后存入数据库并返回短 ID，供大模型在后续计算中直接引用。

---

## 二、 目录结构设计

借鉴企业级 FastAPI 项目规范，实现 MCP 协议、网络路由与底层化学计算业务的彻底解耦：

```text
laspai_mcp_server/
├── core/                       # 核心基础设施层
│   ├── __init__.py
│   ├── config.py               # 全局配置 (环境读取)
│   ├── database.py             # 数据库连接池配置
│   ├── models.py               # 数据库 ORM 模型定义 (完美保留，非常重要)
│   ├── logger.py               # 结合 contextvars 的上下文日志配置
│   └── security.py             # MCP Dependencies 鉴权逻辑 (原先的 token_verifier)
├── mcp_handlers/               # MCP 协议核心实现层
│   ├── __init__.py
│   ├── server.py               # 实例化 MCPServer，挂载 tools 与 resources
│   ├── resources.py            # @mcp.resource 实现
│   └── tools/                  # @mcp.tool 实现目录
│       ├── __init__.py
│       └── organic_tools.py    # 具体领域的计算工具
├── services/                   # 纯业务逻辑层
│   ├── __init__.py
│   └── chemistry_client.py     # 封装与底层高算力集群的交互
├── logs/                       # 运行时日志目录
│   └── mcp_server.log          # 自动注入 user_id 的追踪日志
├── __init__.py
├── main.py                     # 主入口：支持 SSE 和 STDIO 启动
├── requirements.txt            
└── .env
```