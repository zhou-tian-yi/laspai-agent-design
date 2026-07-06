# LASPAI MCP Server 设计方案

本文档概述了将计算化学智能体底层工具调用迁移至 MCP (Model Context Protocol) 架构的服务器端功能需求与目录结构设计。

---

## 一、 核心功能架构

### 1. 基础设施与传输层 (Infrastructure & Transport)
* **平行挂载架构**：采用 FastAPI 作为主应用宿主，负责处理全局中间件、CORS 以及原生的 REST API。基于 Starlette 的原生 `MCPServer` 实例作为独立的 ASGI 子应用，通过 `Mount` 整体挂载至 `/mcp` 路径，实现标准 Web API 与 MCP 协议的底层解耦。
* **全链路追踪**：智能接力上游 `traceparent` Header——智能体端发来请求时继承其 `trace_id` 创建子 Span；第三方 REST API 直接调用时由 OTel 自动生成全新 Root Span。Span Attributes 绑定 `user_id`（从 Token 解析）、`tool_name`、`job_id`（计算任务维度）。
* **统一日志管理 (Logging)**：配置 OTel 将 trace/span 上下文桥接到 Python `logging` 模块，通过自定义 Log Formatter 将 `trace_id`、`span_id`、`user_id` 自动注入到每行日志中，持久化记录 API 调用、鉴权状态、工具执行耗时及内部异常。
* **无状态上下文鉴权**：采用无状态 Token 校验机制。要求客户端在 HTTP Header 中携带鉴权 Token。
* **SSE 路由与底层挂载接管**：使用 MCP SDK 的 `SseServerTransport` 接管传输层，针对两种接口采取不同的挂载策略：
  * **GET `/mcp/sse`**：采用 FastAPI 原生路由定义，用于建立及保持 Server-Sent Events 长连接流。
  * **POST `/mcp/messages`**：鉴于 MCP SDK 的底层设计，绕过 FastAPI 路由层，直接使用 Starlette 的 `Mount` 挂载 `sse.handle_post_message` ASGI 应用。
* **ASGI 级鉴权拦截器 (Auth Wrapper)**：针对上述通过 `Mount` 挂载的 `/mcp/messages` 端点，编写自定义 ASGI 包装器进行拦截。在请求进入 MCP SDK 处理前，提取 Header 并校验 Token，将解析出的 `user_id` 实时注入当前独立请求的 `contextvars` 异步上下文中，供后方的 `@mcp.tool` 和 `@mcp.resource` 安全读取，实现彻底的多租户权限隔离。

### 2. 数据持久化层 (Database & Storage)
* **结构入库与权限绑定**：设计统一的数据库表（如 `ChemistryDataModel`），将化学结构文件（PDB/CIF 文本）、元数据以及所属的 `user_id` 统一持久化存储。
* **短 ID 生成机制**：舍弃长 UUID，生成安全的、带语义前缀的短 ID（如 `mol_8f3a9b`、`crys_2b19c`）作为大模型在工具链中传递的参数，防范大模型幻觉与 Token 浪费。

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
项目根目录/
├── src/
│   └── laspai_mcp_server/
│       ├── __init__.py
│       ├── main.py                     # 应用主入口：组装 FastAPI 实例与挂载 MCP 底层应用
│       ├── core/                       # 核心基础设施层
│       │   ├── __init__.py
│       │   ├── config.py               # 全局配置中心 (数据库URI、计算后端URL等)
│       │   ├── database.py             # 数据库引擎与 SQLModel 数据表定义
│       │   ├── security.py             # 核心鉴权逻辑 (提供给 MCPServer 与 FastAPI 依赖)
│       │   ├── middleware.py           # OTel 解析及全局请求中间件
│       │   └── logger.py               # 统一日志格式与分发配置
│       ├── api/                        # FastAPI 独立 REST 路由层
│       │   ├── __init__.py
│       │   ├── deps.py                 # FastAPI 专用的依赖注入函数
│       │   └── upload_routes.py        # /api/v1/files 等独立业务接口
│       ├── mcp_handlers/               # MCP 协议核心实现层
│       │   ├── __init__.py
│       │   ├── server.py               # MCPServer 实例初始化、鉴权注册与 asgi_app 导出
│       │   ├── resources.py            # @mcp.resource 资源读取注册
│       │   └── tools/                  # @mcp.tool 工具链注册 (按化学领域解耦)
│       │       ├── __init__.py
│       │       ├── molecule_tools.py   # 小分子生成与优化工具
│       │       └── crystal_tools.py    # 晶体生成与体相优化工具
│       └── services/                   # 纯业务逻辑层 (供 MCP Tools 调用，隔离外部协议)
│           ├── __init__.py
│           └── chemistry_client.py     # 封装与底层计算集群的 HTTP 请求与重试机制
├── logs/                               # 运行时产生的日志文件存储目录
├── requirements.txt                    # 项目依赖管理文件
└── .env                                # 本地环境变量配置文件
```

抱歉，你说得完全正确。作为一份正式的设计文档，应该具备清晰、独立和客观的陈述，不该残留架构演进过程中的讨论痕迹。同时，目录结构也应当准确反映你实际的 `src/laspai_mcp_server/` 代码库组织方式。

我已经为你重新编写了一份干净、标准且符合当前最佳实践的全新设计文档：

# LASPAI MCP Server 设计方案

本文档概述了计算化学智能体底层工具调用的 MCP (Model Context Protocol) 架构服务器端功能需求与目录结构设计。本系统基于 FastAPI 与原生 MCPServer 融合的平行架构进行构建。

---

## 一、 核心功能架构

### 1. 基础设施与传输层 (Infrastructure & Transport)

* **平行挂载架构**：采用 FastAPI 作为主应用宿主，负责处理全局中间件、CORS 以及原生的 REST API。基于 Starlette 的原生 `MCPServer` 实例作为独立的 ASGI 子应用，通过 `Mount` 整体挂载至 `/mcp` 路径，实现标准 Web API 与 MCP 协议的底层解耦。
* **原生无状态鉴权**：利用 `MCPServer` 官方提供的 `token_verifier` 接口实现无状态鉴权。大模型客户端在建立连接或发送请求时携带 Token，由 SDK 内部机制自动完成校验、拦截与身份解析。
* **显式上下文传递**：在所有的 MCP 核心业务逻辑中，通过官方 SDK 传入的 `Context` 对象直接提取经过校验的 `user_id`，实现安全、可靠且无歧义的多租户权限隔离。
* **全链路追踪**：智能接力上游 `traceparent` Header。智能体端发来请求时继承其 `trace_id` 创建子 Span；独立 REST API 调用时由 OTel 自动生成 Root Span。Span Attributes 统一绑定提取的 `user_id`、`tool_name` 以及 `job_id`。
* **统一日志管理**：配置 OTel 将 trace/span 上下文桥接到 Python `logging` 模块。通过自定义 Log Formatter 自动将 `trace_id`、`span_id`、`user_id` 注入到每行日志中，实现系统运行状态的持久化记录与精准溯源。

### 2. 数据持久化层 (Database & Storage)

* **结构入库与权限绑定**：设计统一的数据库表（如 `ArtifactModel`），将化学结构文件（PDB/CIF 文本）、计算元数据以及所属的 `user_id` 统一持久化存储。
* **短 ID 生成机制**：生成安全的、带语义前缀的短 ID（如 `mol_8f3a9b`、`crys_2b19c`）作为大模型在工具链中传递的主键参数，以此防范大模型幻觉并节约 Token 消耗。

### 3. MCP 核心能力 (Capabilities)

* **计算工具 (`@mcp.tool`)**：接收大模型传来的短 ID，结合 SDK 显式注入的 `Context` 中的 `user_id` 进行数据权限校验。从数据库安全提取结构后，调用底层的化学计算集群 API（如生成、优化、振动分析），计算完成后仅向大模型返回结构化的结果元数据和新的短 ID。
* **资源读取 (`@mcp.resource`)**：注册标准的资源 URI 模板（如 `artifact://{resource_type}/{primary_key}`）。当客户端请求该 URI 时，服务器解析主键并结合 `Context` 中的 `user_id` 进行数据库比对，鉴权通过后直接返回化学结构的纯文本内容。

### 4. 混合 API (Custom Routes)

* **独立文件上传接口**：提供标准的 `POST /api/v1/files/upload` 接口，充分利用 FastAPI 的 Pydantic 校验与可视化 OpenAPI 文档。该接口接收前端用户上传的结构文件，存入数据库并返回短 ID，供大模型在后续步骤中直接引用。

---

## 二、 目录结构设计

项目采用标准的 `src` 布局规范，将底层逻辑、REST API 与 MCP 协议处理进行彻底分离，保障企业级项目的可维护性：

```text
项目根目录/
├── src/
│   └── laspai_mcp_server/
│       ├── __init__.py
│       ├── main.py                     # 应用主入口：组装 FastAPI 实例与挂载 MCP 底层应用
│       ├── core/                       # 核心基础设施层
│       │   ├── __init__.py
│       │   ├── config.py               # 全局配置中心 (数据库URI、计算后端URL等)
│       │   ├── database.py             # 数据库引擎与 SQLModel 数据表定义
│       │   ├── security.py             # 核心鉴权逻辑 (提供给 MCPServer 与 FastAPI 依赖)
│       │   ├── middleware.py           # OTel 解析及全局请求中间件
│       │   └── logger.py               # 统一日志格式与分发配置
│       ├── api/                        # FastAPI 独立 REST 路由层
│       │   ├── __init__.py
│       │   ├── deps.py                 # FastAPI 专用的依赖注入函数
│       │   └── upload_routes.py        # /api/v1/files 等独立业务接口
│       ├── mcp_handlers/               # MCP 协议核心实现层
│       │   ├── __init__.py
│       │   ├── server.py               # MCPServer 实例初始化、鉴权注册与 asgi_app 导出
│       │   ├── resources.py            # @mcp.resource 资源读取注册
│       │   └── tools/                  # @mcp.tool 工具链注册 (按化学领域解耦)
│       │       ├── __init__.py
│       │       ├── molecule_tools.py   # 小分子生成与优化工具
│       │       └── crystal_tools.py    # 晶体生成与体相优化工具
│       └── services/                   # 纯业务逻辑层 (供 MCP Tools 调用，隔离外部协议)
│           ├── __init__.py
│           └── chemistry_client.py     # 封装与底层计算集群的 HTTP 请求与重试机制
├── logs/                               # 运行时产生的日志文件存储目录
├── requirements.txt                    # 项目依赖管理文件
└── .env                                # 本地环境变量配置文件

```