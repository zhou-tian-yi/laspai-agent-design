# LASPAI MCP Server 设计方案

本文档定义了计算化学智能体底层工具调用服务器的架构设计。本系统基于官方原生的 Model Context Protocol (MCP) SDK 构建，通过规范化的底层设施提供安全、高效的化学计算服务。

---

## 一、 核心架构与基础设施

* **原生 SDK 驱动**：全面采用官方 MCP SDK，利用其内建的 Context、Dependencies 和 Lifespan 管理核心业务逻辑与服务器生命周期。
* **融合部署模式**：利用官方支持的 Add to an existing app 特性，保留 FastAPI 作为宿主框架（Host）挂载 MCP 应用。FastAPI 负责处理独立的 REST API，MCP Server 专职处理智能体的 JSON-RPC 请求。
* **原生鉴权拦截**：应用 MCP SDK 原生的 Authorization 机制，在客户端建立连接时自动完成 Token 校验与拦截。
* **显式上下文隔离**：在工具与资源处理器中，直接通过官方 SDK 注入的 Context 对象安全获取 `user_id`，实现严格的多租户数据隔离。
* **全链路追踪体系**：原生集成 SDK 支持的 OpenTelemetry 特性。系统自动接力上游 `traceparent`，并将链路 ID 与用户身份无缝注入统一日志系统。

---

## 二、 数据持久化与流转

* **统一存储模型**：设计标准化的数据库表（如 `chemistry_data`），将 PDB/CIF 等大体积结构文本、物理元数据及所属 `user_id` 进行结构化持久存储。
* **短 ID 引用机制**：生成安全的、带语义前缀的短 ID（如 `mol_8f3a9b`）作为大模型交互的主键。在连续的工具调用中仅传递短 ID，有效降低 Token 消耗并遏制大模型幻觉。
* **双轨数据入口**：支持智能体直接通过 MCP 工具传入结构字符串；同时保留 REST API 端点供外部系统或前端直传大体积文件并换取短 ID。

---

## 三、 MCP 核心能力矩阵

* **化学计算工具 (`@mcp.tool`)**：接收短 ID，结合 Context 校验权限后，调度底层化学计算集群（生成、构型优化等），计算完成后向智能体返回状态与新生成的短 ID。
* **结果动态寻址 (`@mcp.resource`)**：注册 `artifact://{resource_type}/{short_id}` 格式的 URI 模板。授权智能体按需读取结构文件的纯文本内容，避免庞大坐标数据污染大模型对话上下文。

---

## 二、 目录结构设计

借鉴企业级 FastAPI 项目规范，实现 MCP 协议、网络路由与底层化学计算业务的彻底解耦：

```text
laspai_mcp_server/
├── core/                       # 核心基础设施层
│   ├── __init__.py
│   ├── config.py               # 全局配置 (环境读取)
│   ├── database.py             # 数据库连接池配置
│   ├── models.py               # 数据库 ORM 模型定义
│   ├── logger.py               # 结合 contextvars 的上下文日志配置
│   ├── security.py             # MCP Dependencies 鉴权逻辑
│   └── context.py              # 上下文变量（用于日志）
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