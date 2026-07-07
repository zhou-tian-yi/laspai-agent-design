# Dify 数据持久化架构分析

> 基于 Dify 开源项目（[GitHub](https://github.com/langgenius/dify)）源码分析
> 分析日期：2026-07-07

---

## 目录

1. [一、总体技术栈](#一总体技术栈)
2. [二、用户系统](#二用户系统)
3. [三、权限管理](#三权限管理)
4. [四、对话数据存储与恢复](#四对话数据存储与恢复)
5. [五、文件管理系统](#五文件管理系统)
6. [六、用量计费系统](#六用量计费系统)
7. [七、设计模式总结](#七设计模式总结)

---

## 一、总体技术栈

| 层面 | 技术选择 |
|------|---------|
| **数据库** | PostgreSQL（主选）/ MySQL（兼容），通过自定义类型系统屏蔽差异 |
| **ORM** | SQLAlchemy + Flask-SQLAlchemy |
| **迁移** | Alembic（100+ 迁移文件） |
| **缓存** | Redis（速率限制、配额锁、API Token缓存、单次查询合并） |
| **文件存储** | OpenDAL / S3 / Azure Blob / 阿里云 OSS / 本地 FS |
| **任务队列** | Celery + Redis（异步删除、名称生成等） |

### 1.1 数据库配置

- `DB_TYPE`: postgresql（默认）/ mysql / oceanbase / seekdb
- 连接池：`POOL_SIZE=30`, `MAX_OVERFLOW=10`, `POOL_RECYCLE=3600`
- 默认时区：UTC

### 1.2 命名约定

```
ix_   = %(column_0_label)s_idx         -- 索引
uq_   = %(table_name)s_%(column_0_name)s_key   -- 唯一约束
fk_   = %(table_name)s_%(column_0_name)s_fkey  -- 外键
pk_   = %(table_name)s_pkey            -- 主键
```

### 1.3 ORM 基类

```python
# 两种基类并存

class Base:          # 传统 SQLAlchemy → Conversation, Message 等旧模型
    pass

class TypeBase:      # MappedAsDataclass → Agent, Trigger 等新模型
    pass

# 默认字段混入
class DefaultFieldsMixin:
    id = Column(StringUUID, primary_key=True, default=gen_uuidv7)  # uuidv7 可排序
    created_at = Column(DateTime, server_default=func.current_timestamp())
    updated_at = Column(DateTime, onupdate=func.current_timestamp())
```

---

## 二、用户系统

### 2.1 架构：多租户 + 租户内多角色

```
Account (全局用户) ──┐
                     ├── TenantAccountJoin (成员关系 + 角色 + 当前空间)
Tenant (工作空间) ───┘
```

一个用户可以加入多个工作空间，每个工作空间有独立角色，`current` 标志当前活跃的工作空间。

### 2.2 核心表 DDL

```sql
-- ============================================
-- 用户账户表
-- ============================================
CREATE TABLE accounts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    email               VARCHAR(255),
    password            VARCHAR(255),            -- 加密存储
    password_salt       VARCHAR(255),            -- 盐值
    avatar              VARCHAR(255),
    interface_language  VARCHAR(255),
    interface_theme     VARCHAR(255),
    timezone            VARCHAR(255),
    last_login_at       TIMESTAMP,
    last_login_ip       VARCHAR(255),
    last_active_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    status              VARCHAR(16) DEFAULT 'active',  -- active | banned | pending
    initialized_at      TIMESTAMP,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX account_email_idx ON accounts(email);

-- ============================================
-- 工作空间（租户）表
-- ============================================
CREATE TABLE tenants (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    encrypt_public_key  TEXT,
    plan                VARCHAR(255) DEFAULT 'basic',  -- sandbox | professional | team
    status              VARCHAR(16) DEFAULT 'normal',  -- normal | archive
    custom_config       TEXT,               -- JSON
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- ============================================
-- 成员关系表（核心：承载角色和当前空间标记）
-- ============================================
CREATE TABLE tenant_account_joins (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    current         BOOLEAN DEFAULT FALSE,       -- 当前活跃工作空间
    role            VARCHAR(20) DEFAULT 'normal',
        -- owner | admin | editor | normal | dataset_operator
    invited_by      UUID,
    last_opened_at  TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, account_id)
);
CREATE INDEX idx_join_account_id ON tenant_account_joins(account_id);
CREATE INDEX idx_join_tenant_id  ON tenant_account_joins(tenant_id);

-- ============================================
-- 第三方登录绑定表
-- ============================================
CREATE TABLE account_integrates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL,
    provider        VARCHAR(16) NOT NULL,        -- google | github | ...
    open_id         VARCHAR(255) NOT NULL,
    encrypted_token VARCHAR(255),
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(account_id, provider),
    UNIQUE(provider, open_id)
);

-- ============================================
-- 邀请码表
-- ============================================
CREATE TABLE invitation_codes (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    batch               VARCHAR(255),
    code                VARCHAR(32),
    status              VARCHAR(16) DEFAULT 'unused',  -- unused | used
    used_at             TIMESTAMP,
    used_by_tenant_id   UUID,
    used_by_account_id  UUID,
    deprecated_at       TIMESTAMP,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_code_batch ON invitation_codes(batch);
CREATE INDEX idx_code_code  ON invitation_codes(code, status);
```

### 2.3 角色枚举

| 角色 | 值 | 说明 |
|------|-----|------|
| OWNER | `owner` | 工作空间所有者（创建者） |
| ADMIN | `admin` | 管理员，可管理成员 |
| EDITOR | `editor` | 编辑者，可编辑应用和数据集 |
| NORMAL | `normal` | 普通成员（只读） |
| DATASET_OPERATOR | `dataset_operator` | 数据集操作员 |

### 2.4 设计要点

1. **密码**：加盐哈希存储，`password` + `password_salt` 两个字段
2. **空间切换**：通过 `tenant_account_joins.current = TRUE` 标记当前工作空间
3. **角色枚举**：Python 层 `StrEnum`，数据库存 VARCHAR
4. **邀请码**：支持批量生成和过期

---

## 三、权限管理

### 3.1 双层权限体系

通过配置开关 `RBAC_ENABLED` 切换两套系统：

#### 模式 A：基于角色的权限（默认，RBAC_ENABLED=False）

5 个预定义角色，硬编码权限映射：

```python
class TenantAccountRole(StrEnum):
    OWNER = "owner"
    ADMIN = "admin"
    EDITOR = "editor"
    NORMAL = "normal"
    DATASET_OPERATOR = "dataset_operator"

    @staticmethod
    def is_editing_role(role):
        return role in (OWNER, ADMIN, EDITOR)

    @staticmethod
    def is_dataset_edit_role(role):
        return role in (OWNER, ADMIN, EDITOR, DATASET_OPERATOR)
```

| 角色 | 可编辑 | 可管理成员 | 可管理数据集 |
|------|--------|-----------|-------------|
| `owner` | ✅ | ✅ | ✅ |
| `admin` | ✅ | ✅ | ✅ |
| `editor` | ✅ | ❌ | ✅ |
| `dataset_operator` | ❌ | ❌ | ✅ |
| `normal` | ❌ | ❌ | ❌ |

#### 模式 B：RBAC 细粒度权限（RBAC_ENABLED=True）

60+ 个权限点，按资源分类，通过外部企业版 HTTP API 校验：

```python
RBACPermission = {
    "WORKSPACE": [
        "WORKSPACE_MEMBER_MANAGE", "WORKSPACE_ROLE_MANAGE",
        "API_EXTENSION_MANAGE", "CUSTOMIZATION_MANAGE"
    ],
    "APP": [
        "APP_EDIT", "APP_DELETE", "APP_CREATE_AND_MANAGEMENT",
        "APP_RELEASE_AND_VERSION", "APP_LOG_AND_ANNOTATION",
        "APP_VIEW_LAYOUT", "APP_ACCESS_CONFIG", ...
    ],
    "DATASET": [
        "DATASET_EDIT", "DATASET_DELETE", "DATASET_CREATE_AND_MANAGEMENT",
        "DATASET_API_KEY_MANAGE", "DATASET_USE", ...
    ],
    "PLUGIN": [
        "PLUGIN_INSTALL", "PLUGIN_MANAGE", "PLUGIN_DEBUG", ...
    ],
    "CREDENTIAL": [
        "CREDENTIAL_USE", "CREDENTIAL_CREATE", "CREDENTIAL_MANAGE"
    ],
}
```

### 3.2 数据级权限

```sql
-- 数据集行级权限
CREATE TABLE dataset_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id      UUID NOT NULL,
    account_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    has_permission  BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON dataset_permissions(dataset_id);
CREATE INDEX ON dataset_permissions(account_id);
CREATE INDEX ON dataset_permissions(tenant_id);

-- 凭证访问权限（泛型设计，用 credential_type 区分多种凭证）
CREATE TABLE credential_permissions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    credential_id    UUID NOT NULL,
    credential_type  VARCHAR(40) NOT NULL,  -- trigger_subscription | builtin_tool_provider | ...
    account_id       UUID NOT NULL,
    tenant_id        UUID NOT NULL,
    has_permission   BOOLEAN DEFAULT TRUE,
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON credential_permissions(account_id);
CREATE INDEX ON credential_permissions(tenant_id);
CREATE INDEX ON credential_permissions(credential_id, credential_type);
```

### 3.3 插件权限配置

```sql
-- 插件安装和调试权限（工作空间级别配置）
CREATE TABLE account_plugin_permissions (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID UNIQUE NOT NULL,
    install_permission   VARCHAR(16) DEFAULT 'admins',  -- everyone | admins | noone
    debug_permission     VARCHAR(16) DEFAULT 'everyone'  -- everyone | admins | noone
);
```

### 3.4 鉴权框架：5 层 API 外观

| 装饰器文件 | 适用范围 | 鉴权方式 |
|-----------|---------|---------|
| `console/wraps.py` | 管理后台 API | Flask-Login Session + 角色检查 |
| `service_api/wraps.py` | 开放 API | Bearer Token（App/Dataset API Key） |
| `web/wraps.py` | 嵌入式 Web 应用 | JWT（Passport 签发） |
| `inner_api/wraps.py` | 内部微服务 | X-Inner-Api-Key Header |
| `common/wraps.py` | 通用 RBAC | POST 到企业版 `/rbac/check-access` |

**典型装饰器栈**：
```python
# Console API 的一个典型端点
@setup_required
@login_required
@account_initialization_required
@rbac_permission_required(RBACResourceScope.APP, RBACPermission.APP_EDIT)
def post(self, app_id):
    ...
```

---

## 四、对话数据存储与恢复

### 4.1 核心表 DDL

```sql
-- ============================================
-- 应用表（每个应用是对话的容器）
-- ============================================
CREATE TABLE apps (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id             UUID NOT NULL,
    name                  VARCHAR(255) NOT NULL,
    description           TEXT DEFAULT '',
    mode                  VARCHAR(50) NOT NULL,
        -- completion | workflow | chat | advanced-chat | agent-chat | agent | channel | rag-pipeline
    icon_type             VARCHAR(16),
    icon                  VARCHAR(255),
    icon_background       VARCHAR(255),
    app_model_config_id   UUID,
    workflow_id           UUID,
    status                VARCHAR(16) DEFAULT 'normal',
    enable_site           BOOLEAN,
    enable_api            BOOLEAN,
    api_rpm               INT DEFAULT 0,       -- 每分钟请求数限制
    api_rph               INT DEFAULT 0,       -- 每小时请求数限制
    max_active_requests   INT,                 -- 最大并发请求
    is_demo               BOOLEAN DEFAULT FALSE,
    is_public             BOOLEAN DEFAULT FALSE,
    created_by            UUID,
    maintainer            UUID,                -- 应用维护者（用于权限校验）
    created_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at            TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX app_tenant_id_idx         ON apps(tenant_id);
CREATE INDEX app_tenant_maintainer_idx ON apps(tenant_id, maintainer);

-- ============================================
-- 应用模型配置表（JSON 灵活存储）
-- ============================================
CREATE TABLE app_model_configs (
    id                               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id                           UUID NOT NULL,
    provider                         VARCHAR(255),
    model_id                         VARCHAR(255),
    configs                          JSON,
    opening_statement                TEXT,
    suggested_questions              TEXT,
    suggested_questions_after_answer TEXT,
    speech_to_text                   TEXT,
    text_to_speech                   TEXT,
    more_like_this                   TEXT,
    model                            TEXT,          -- JSON
    user_input_form                  TEXT,          -- JSON
    pre_prompt                       TEXT,
    agent_mode                       TEXT,          -- JSON
    sensitive_word_avoidance         TEXT,
    retriever_resource               TEXT,
    prompt_type                      VARCHAR(16) DEFAULT 'simple',  -- simple | advanced
    chat_prompt_config               TEXT,
    completion_prompt_config         TEXT,
    dataset_configs                  TEXT,          -- JSON
    external_data_tools              TEXT,          -- JSON
    file_upload                      TEXT,          -- JSON
    dataset_query_variable           VARCHAR(255),
    created_by                       UUID,
    created_at                       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by                       UUID,
    updated_at                       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON app_model_configs(app_id);

-- ============================================
-- 对话（会话）表
-- ============================================
CREATE TABLE conversations (
    id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id                 UUID NOT NULL,
    app_model_config_id    UUID,
    model_provider         VARCHAR(255),
    model_id               VARCHAR(255),
    override_model_configs TEXT,          -- JSON，调试模式的模型参数覆盖
    mode                   VARCHAR(50) NOT NULL,
    name                   VARCHAR(255),
    summary                TEXT,
    inputs                 JSONB,         -- 用户输入变量（含文件兼容桥）
    introduction           TEXT,
    system_instruction     TEXT,
    system_instruction_tokens INT DEFAULT 0,
    status                 VARCHAR(16) DEFAULT 'normal',
    invoke_from            VARCHAR(50),   -- service-api | web-app | explore | debugger
    from_source            VARCHAR(16),   -- api | console
    from_end_user_id       UUID,
    from_account_id        UUID,
    read_at                TIMESTAMP,     -- 未读标记
    read_account_id        UUID,
    dialogue_count         INT DEFAULT 0,
    is_deleted             BOOLEAN DEFAULT FALSE,  -- 软删除
    created_at             TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at             TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 关键索引
CREATE INDEX conversation_app_from_user_idx
    ON conversations(app_id, from_source, from_end_user_id);
CREATE INDEX conversation_app_created_at_idx
    ON conversations(app_id, created_at DESC) WHERE is_deleted IS FALSE;
CREATE INDEX conversation_app_updated_at_idx
    ON conversations(app_id, updated_at DESC) WHERE is_deleted IS FALSE;

-- ============================================
-- 消息表（核心：承载对话历史和 token 计费）
-- ============================================
CREATE TABLE messages (
    id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id                   UUID NOT NULL,
    model_provider           VARCHAR(255),
    model_id                 VARCHAR(255),
    override_model_configs   TEXT,            -- JSON
    conversation_id          UUID NOT NULL REFERENCES conversations(id),
    inputs                   JSONB,
    query                    TEXT,            -- 用户问题
    message                  JSONB,           -- 发送给 LLM 的完整 prompt 结构

    -- Token 计费字段
    message_tokens           INT DEFAULT 0,
    message_unit_price       NUMERIC(10,4),
    message_price_unit       NUMERIC(10,7) DEFAULT 0.001,
    answer                   TEXT,            -- LLM 回复
    answer_tokens            INT DEFAULT 0,
    answer_unit_price        NUMERIC(10,4),
    answer_price_unit        NUMERIC(10,7) DEFAULT 0.001,
    total_price              NUMERIC(10,7),
    currency                 VARCHAR(255),
    provider_response_latency FLOAT DEFAULT 0, -- 秒

    -- 状态
    parent_message_id        UUID,            -- 回复链
    status                   VARCHAR(16) DEFAULT 'normal',
    error                    TEXT,
    message_metadata         TEXT,            -- JSON
    agent_based              BOOLEAN DEFAULT FALSE,
    workflow_run_id          UUID,
    app_mode                 VARCHAR(50),

    -- 来源追踪
    invoke_from              VARCHAR(50),
    from_source              VARCHAR(16),
    from_end_user_id         UUID,
    from_account_id          UUID,

    created_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at               TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX message_conversation_id_idx ON messages(conversation_id);
CREATE INDEX message_app_id_idx          ON messages(app_id, created_at);
CREATE INDEX message_end_user_idx        ON messages(app_id, from_source, from_end_user_id);
CREATE INDEX message_account_idx         ON messages(app_id, from_source, from_account_id);
CREATE INDEX message_created_at_id_idx   ON messages(created_at, id);

-- ============================================
-- 消息反馈表（点赞/踩）
-- ============================================
CREATE TABLE message_feedbacks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id          UUID NOT NULL,
    conversation_id UUID NOT NULL,
    message_id      UUID NOT NULL,
    rating          VARCHAR(16),              -- like | dislike
    from_source     VARCHAR(16),              -- user | admin
    content         TEXT,
    from_end_user_id UUID,
    from_account_id  UUID,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON message_feedbacks(app_id);
CREATE INDEX ON message_feedbacks(message_id, from_source);
CREATE INDEX ON message_feedbacks(conversation_id, from_source, rating);

-- ============================================
-- 消息文件关联表
-- ============================================
CREATE TABLE message_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL,
    type            VARCHAR(16),          -- image | document | audio | video | custom
    transfer_method VARCHAR(16),          -- local_file | remote_url | tool_file
    created_by_role VARCHAR(16),          -- account | end_user
    created_by      UUID,
    belongs_to      VARCHAR(16),          -- user | assistant
    url             TEXT,
    upload_file_id  UUID,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON message_files(message_id);
CREATE INDEX ON message_files(created_by);

-- ============================================
-- Agent 思考链（工具调用追踪）
-- ============================================
CREATE TABLE message_agent_thoughts (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id         UUID NOT NULL,
    message_chain_id   UUID,
    position           INT,
    thought            TEXT,              -- 思考过程
    tool               TEXT,              -- 工具名
    tool_labels_str    TEXT,
    tool_meta_str      TEXT,
    tool_input         TEXT,              -- 工具输入 JSON
    observation        TEXT,              -- 工具输出
    tool_process_data  TEXT,
    message            TEXT,
    message_token      INT,
    message_unit_price NUMERIC(10,4),
    message_price_unit NUMERIC(10,7),
    message_files      TEXT,
    answer             TEXT,
    answer_token       INT,
    answer_unit_price  NUMERIC(10,4),
    answer_price_unit  NUMERIC(10,7),
    tokens             INT,
    total_price        NUMERIC(10,7),
    currency           VARCHAR(255),
    latency            FLOAT,
    created_by_role    VARCHAR(16),
    created_by         UUID,
    created_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON message_agent_thoughts(message_id);
CREATE INDEX ON message_agent_thoughts(message_chain_id);

-- ============================================
-- 消息标注（人工标注 QA 对）
-- ============================================
CREATE TABLE message_annotations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id          UUID NOT NULL,
    conversation_id UUID,
    message_id      UUID,
    question        TEXT,
    content         TEXT,
    hit_count       INT DEFAULT 0,
    account_id      UUID,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON message_annotations(app_id);
CREATE INDEX ON message_annotations(message_id);

-- ============================================
-- 书签/收藏
-- ============================================
CREATE TABLE pinned_conversations (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id           UUID,
    conversation_id  UUID,
    created_by_role  VARCHAR(16),
    created_by       UUID,
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON pinned_conversations(app_id, conversation_id, created_by_role, created_by);

CREATE TABLE saved_messages (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id           UUID,
    message_id       UUID,
    created_by_role  VARCHAR(16),
    created_by       UUID,
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON saved_messages(app_id, message_id, created_by_role, created_by);
```

### 4.2 对话生命周期

```
用户提问
  │
  ▼
_init_generate_records()
  ├── 创建/复用 Conversation 行
  ├── 创建 Message 行（answer 为空，token=0）
  ├── 创建 MessageFile 行
  └── commit
  │
  ▼
_stream_generate() ← LLM streaming response
  │
  ├── QueueLLMChunkEvent → 累积 answer 文本
  ├── QueueRetrieverResourcesEvent → 记录检索资源
  ├── QueueAnnotationReplyEvent → 记录标注命中
  │
  ▼
_save_message() （完成或出错时触发）
  ├── 填充 message.message（prompt 结构化数据）
  ├── 填充 answer + token 计数 + 价格
  ├── 设置 message_metadata
  └── 触发信号 message_was_created
  │
  ▼
消息持久化完成
```

### 4.3 对话历史恢复（Token Buffer Memory）

```python
# 核心逻辑（core/memory/token_buffer_memory.py）
class TokenBufferMemory:
    def get_history_prompt_messages(self):
        # 1. 查询当前对话所有消息（倒序，最多 500 条）
        messages = db.query(Message)
            .filter(conversation_id=self.conv_id)
            .order_by(Message.created_at.desc())
            .limit(500)

        # 2. 按 parent_message_id 还原线程结构
        thread = extract_thread_messages(messages)

        # 3. 跳过空 answer（刚创建未完成的消息）
        thread = [m for m in thread if m.answer]

        # 4. 批量加载文件（2 次查询防 N+1）
        user_files = batch_load(message_ids, belongs_to='user')
        assistant_files = batch_load(message_ids, belongs_to='assistant')

        # 5. 构建 UserPromptMessage / AssistantPromptMessage
        prompt_messages = build_prompt_messages(thread, user_files, assistant_files)

        # 6. 按 token 上限裁剪（从最早消息开始移除）
        return trim_by_token_limit(prompt_messages, max_token_limit)
```

### 4.4 Workflow 暂停与恢复（长时间运行 AI 状态恢复）

```
状态外存（WorkflowPause）：
  state_object_key → 外部存储 → GraphEngine 序列化状态

暂停流程：
  工作流遇到 pause 节点（如等待人工输入）
  → GraphEngine 运行时状态序列化 → 存入外部存储
  → state_object_key 记录到 workflow_pauses 表
  → 等待外部事件触发

恢复流程：
  人工提交表单 / 定时器触发
  → 按 key 读取序列化状态
  → 反序列化恢复 GraphEngine
  → 设置 resumed_at（pause 记录保留为审计追踪）
```

```sql
-- 工作流暂停表
CREATE TABLE workflow_pauses (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id      UUID,
    workflow_run_id  UUID UNIQUE,
    state_object_key VARCHAR(255),        -- 外部存储 key
    resumed_at       TIMESTAMP,           -- 恢复时间
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE workflow_pause_reasons (
    id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pause_id UUID,
    type     VARCHAR(32),    -- hitl_required | scheduled_pause | legacy_human_input
    form_id  VARCHAR(36),
    message  VARCHAR(255),
    node_id  VARCHAR(255)
);
CREATE INDEX ON workflow_pause_reasons(pause_id);
```

### 4.5 对话变量实时持久化

通过 Graph Engine 的 Event Layer，监听 `NodeRunVariableUpdatedEvent`，立即写入数据库：

```sql
CREATE TABLE workflow_conversation_variables (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id  UUID NOT NULL,
    app_id           UUID NOT NULL,
    data             TEXT,              -- JSON 序列化变量值
    created_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON workflow_conversation_variables(conversation_id);
CREATE INDEX ON workflow_conversation_variables(app_id);
```

### 4.6 软删除与异步清理

```python
# 控制器中同步标记删除
def delete(conversation_id):
    conversation.is_deleted = True
    db.session.commit()
    # 异步清理相关数据
    delete_conversation_related_data.delay(conversation_id)

# Celery 任务（按外键顺序删除）
def delete_conversation_related_data(conversation_id):
    # 顺序：MessageAnnotation → MessageFeedback
    #       → ToolConversationVariables → ToolFile
    #       → ConversationVariable → Message → PinnedConversation
    ...
```

---

## 五、文件管理系统

### 5.1 三层模型结构

```
UploadFile（上传文件元数据）        ToolFile（工具/插件生成文件）
      │                                    │
      └──────── MessageFile（消息关联）──────┘
```

### 5.2 存储路径约定

| 类型 | 存储路径模式 | DB 表 |
|------|-------------|-------|
| 用户上传 | `upload_files/<tenant_id>/<uuid>.<ext>` | `upload_files` |
| 工具/插件生成 | `tools/<tenant_id>/<uuid>.<ext>` | `tool_files` |
| 文本上传 | `upload_files/<tenant_id>/<uuid>.txt` | `upload_files` |

### 5.3 核心表 DDL

```sql
-- ============================================
-- 上传文件主表
-- ============================================
CREATE TABLE upload_files (
    id              UUID PRIMARY KEY,         -- 应用层生成
    tenant_id       UUID NOT NULL,            -- 多租户隔离
    storage_type    VARCHAR(16),              -- s3 | azure-blob | opendal | ...
    key             VARCHAR(255),             -- 存储路径
    name            VARCHAR(255),
    size            INT,
    extension       VARCHAR(255),
    mime_type       VARCHAR(255),
    hash            VARCHAR(255),
    source_url      TEXT,                     -- 远程 URL（若有）
    created_by      UUID,
    created_by_role VARCHAR(16),              -- account | end_user
    used            BOOLEAN DEFAULT FALSE,
    used_by         UUID,
    used_at         TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON upload_files(tenant_id);

-- ============================================
-- 工具文件表（插件/工具生成）
-- ============================================
CREATE TABLE tool_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID,
    tenant_id       UUID NOT NULL,
    conversation_id UUID,
    file_key        VARCHAR(255),
    mimetype        VARCHAR(255),
    original_url    VARCHAR(2048),
    name            VARCHAR(255) DEFAULT '',
    size            INT DEFAULT -1
);
CREATE INDEX ON tool_files(conversation_id);

-- ============================================
-- 人工输入表单文件
-- ============================================
CREATE TABLE human_input_form_upload_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    app_id          UUID,
    form_id         UUID NOT NULL,
    upload_file_id  UUID UNIQUE,
    upload_token_id UUID,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON human_input_form_upload_files(form_id);
```

### 5.4 存储后端抽象

```
BaseStorage (抽象接口)
  ├── save() / load_once() / load_stream()
  ├── download() / exists() / delete()
  └── scan()

  实现类（14 种）：
  ├── S3Storage
  ├── AzureBlobStorage
  ├── GoogleCloudStorage
  ├── AliyunOSSStorage
  ├── TencentCOSStorage
  ├── OpenDALStorage （默认，可切换 fs/s3/gcs/oss）
  ├── SupabaseStorage
  └── ...
```

### 5.5 文件访问安全

**HMAC-SHA256 签名保护所有文件 URL**：

```python
# 签名生成
payload = f"file-preview|{file_id}|{timestamp}|{nonce}"
signature = hmac.new(secret_key, payload.encode(), hashlib.sha256).hexdigest()
url = f"/files/{file_id}/file-preview?timestamp={ts}&nonce={n}&sign={sig}"

# 验证：签名有效 + 300 秒内
FILES_ACCESS_TIMEOUT = 300  # 秒
```

**三层访问控制**：

```python
class FileAccessScope:
    tenant_id                # 始终校验租户隔离
    user_id                  # 所有权校验
    user_from                # ACCOUNT（管理员）或 END_USER（终端用户）
    granted_upload_file_ids  # 执行上下文特批的文件 ID
```

**服务 API 校验链**：
```
请求 → 验证 file_id 存在于 MessageFile → 验证 Message 属于请求的 App
     → UploadFile 存在 → UploadFile.tenant_id == App.tenant_id（租户隔离）
```

### 5.6 文件大小限制

| 文件类型 | 默认限制 |
|---------|---------|
| 图片 | 10 MB |
| 视频 | 100 MB |
| 音频 | 50 MB |
| 文档 | 15 MB |
| 批量上限 | 5 个文件/次 |

### 5.7 文件引用系统

```python
# 通过不透明引用字符串暴露文件 ID，隐藏真实 DB ID
# 格式：dify-file-ref:<base64url-json>
# 内容：{ "record_id": "xxx", "storage_key": "xxx" }

build_file_reference(record_id, storage_key)    # 构建引用
parse_file_reference(ref_string)                # 解析引用
resolve_file_record_id(file_obj)                # 从 File 对象解析 record_id
```

---

## 六、用量计费系统

### 6.1 三层计费架构

```
┌──────────────────────────────────────────────────┐
│ 第一层：Token 计费（Message / AgentThought）      │
│ → 每条消息记录输入/输出 token 数和单价            │
│ → 数据在 messages 和 message_agent_thoughts 表中  │
├──────────────────────────────────────────────────┤
│ 第二层：配额控制（QuotaService）                   │
│ → 触发事件 / API 速率限制                         │
│ → reserve → commit/release 三阶段生命周期          │
│ → 对外部 API 进行 HTTP 调用                       │
├──────────────────────────────────────────────────┤
│ 第三层：积分池（CreditPoolService）                │
│ → 模型调用积分扣减                                │
│ → Redis 分布式锁 + SELECT FOR UPDATE              │
└──────────────────────────────────────────────────┘
```

### 6.2 计费相关表 DDL

```sql
-- ============================================
-- 积分池表（模型调用的额度管理）
-- ============================================
CREATE TABLE tenant_credit_pools (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL,
    pool_type   VARCHAR(16) DEFAULT 'trial',  -- paid | free | trial
    quota_limit BIGINT NOT NULL,
    quota_used  BIGINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON tenant_credit_pools(tenant_id);
CREATE INDEX ON tenant_credit_pools(pool_type);

-- ============================================
-- 模型提供商配额表
-- ============================================
CREATE TABLE providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    provider_name   VARCHAR(255) NOT NULL,
    provider_type   VARCHAR(16) DEFAULT 'custom',
    is_valid        BOOLEAN,
    last_used       TIMESTAMP,
    credential_id   UUID,
    quota_type      VARCHAR(16),          -- paid | free | trial
    quota_limit     BIGINT,
    quota_used      BIGINT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, provider_name, provider_type, quota_type)
);
CREATE INDEX ON providers(tenant_id, provider_name);

-- ============================================
-- 模型订购记录表
-- ============================================
CREATE TABLE provider_orders (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id          UUID NOT NULL,
    provider_name      VARCHAR(255),
    account_id         UUID,
    payment_product_id VARCHAR(191),
    payment_id         VARCHAR(255),
    transaction_id     VARCHAR(255),
    quantity           INT,
    currency           VARCHAR(10),
    total_amount       INT,
    payment_status     VARCHAR(16),       -- wait_pay | paid | failed | refunded
    paid_at            TIMESTAMP,
    pay_failed_at      TIMESTAMP,
    refunded_at        TIMESTAMP,
    created_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at         TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON provider_orders(tenant_id, provider_name);

-- ============================================
-- 速率限制日志表
-- ============================================
CREATE TABLE rate_limit_logs (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL,
    subscription_plan VARCHAR(255),
    operation         VARCHAR(255),      -- knowledge | annotation | ...
    created_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON rate_limit_logs(tenant_id);
CREATE INDEX ON rate_limit_logs(operation);

-- ============================================
-- 试用记录表
-- ============================================
CREATE TABLE trial_apps (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    app_id      UUID UNIQUE NOT NULL,
    tenant_id   UUID NOT NULL,
    trial_limit INT DEFAULT 3,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE account_trial_app_records (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id  UUID NOT NULL,
    app_id      UUID NOT NULL,
    count       INT DEFAULT 0,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(account_id, app_id)
);
```

### 6.3 积分扣减（Redis 分布式锁 + 行锁）

```python
class CreditPoolService:
    def deduct_credits_capped(self, amount):
        lock_key = f"credit_pool:tenant:{tenant_id}:deduct_lock"
        with redis.lock(lock_key):
            pool = (
                select(TenantCreditPool)
                .where(tenant_id=...)
                .with_for_update()       # PostgreSQL 行级锁
            )
            available = pool.quota_limit - pool.quota_used
            to_deduct = min(amount, available)
            pool.quota_used += to_deduct
            db.commit()
            return to_deduct
```

### 6.4 配额三阶段生命周期

```python
class QuotaService:
    def consume(self, quota_type, amount):
        """一次性 reserve + commit（简化的单次调用）"""
        charge = self.reserve(quota_type, amount)
        charge.commit()
        return charge

    def reserve(self, quota_type, amount):
        """预留配额（对外部计费 API 做 HTTP 调用）"""
        # 计费 API 不可达时自动放行为"无限制"
        ...

    def release(self, charge):
        """释放未提交的预留（保证不抛异常）"""
        ...
```

### 6.5 速率限制（Redis 滑动窗口 ZSET）

```python
def rate_limit_check(operation, limit):
    key = f"rate_limit:{tenant_id}:{operation}"
    now = time.time()
    window = 60  # 秒

    with redis.pipeline() as pipe:
        pipe.zremrangebyscore(key, 0, now - window)   # 移除窗口外
        pipe.zadd(key, {str(now): now})                # 添加当前请求
        pipe.zcard(key)                                # 计数
        pipe.expire(key, window)
        _, _, count, _ = pipe.execute()

    if count > limit:
        db.add(RateLimitLog(tenant_id, plan, operation))
        raise RateLimitExceededError()
```

### 6.6 外部化计费服务

```python
class BillingService:
    def get_info(self, tenant_id):
        """获取租户订阅信息"""
        return http_get(f"{BILLING_API}/subscription/{tenant_id}")

    def quota_reserve(self, tenant_id, quota_type, amount):
        """预留 → 返回 reservation_id"""

    def quota_commit(self, reservation_id):
        """确认消耗"""

    def quota_release(self, reservation_id):
        """释放预留"""

# 当 BILLING_ENABLED = False 时，所有调用短路返回"无限制"
```

### 6.7 Feature 聚合服务

```python
class FeatureService:
    def get_features(tenant_id):
        """聚合三层配置"""
        env_config   = load_environment_defaults()
        billing_info = billing_api.get_info(tenant_id)   # 外部计费
        enterprise   = enterprise_api.get_features()      # 企业版
        return merge(env_config, billing_info, enterprise)
```

返回的 `FeatureModel` 包含：
- 成员数上限、应用数上限、向量空间配额
- 知识库速率限制、标注配额
- 文档上传处理配额
- 自定义 Logo、模型负载均衡
- 数据集操作员、拥有者转移
- 触发事件配额、API 速率限制

---

## 七、设计模式总结

### 7.1 核心设计原则

| 原则 | 实现 |
|------|------|
| **多租户隔离** | 所有业务表带 `tenant_id`，所有查询强制租户过滤 |
| **软删除** | `is_deleted` 布尔字段 + 部分索引排除已删除项 |
| **异步清理** | 用户操作同步标记删除，Celery 异步清理关联数据 |
| **UUID 主键** | uuidv7（时间可排序）/ uuidv4，不暴露自增 ID |
| **JSON 灵活存储** | 配置类字段用 TEXT/JSONB，用 property 封装序列化/反序列化 |
| **宽松加载** | relationship 默认 `lazy='raise'` 防 N+1，查询时显式声明 `selectinload` |
| **外部计费** | 计费逻辑抽离为外部 HTTP 服务，支持独立部署或关闭 |
| **文件签名** | 所有文件 URL 带 HMAC-SHA256 签名，300 秒过期 |

### 7.2 自定义 SQLAlchemy 类型

```python
StringUUID:    UUID ↔ VARCHAR(36) / Postgres 原生 UUID
LongText:      TEXT / LONGTEXT（MySQL）
JSONModelColumn[T]: Pydantic 模型 ↔ JSON 文本（自动序列化/反序列化）
BinaryData:    BYTEA / LONGBLOB（pickle 序列化）
AdjustedJSON:  JSONB / JSON + GIN 索引
EnumText[T]:   StrEnum ↔ VARCHAR（自动取长度）
```

### 7.3 索引设计策略

| 场景 | 索引类型 |
|------|---------|
| 租户过滤 + 时间排序 | 复合索引 `(tenant_id, created_at)` |
| 多租户 + 用户级查询 | 复合索引 `(tenant_id, user_id)` |
| 软删除 + 时间排序 | 部分索引 `ON col WHERE is_deleted IS FALSE` |
| JSONB 查询 | GIN 索引 |
| 带关联的复杂查询 | `SELECT FOR UPDATE`（积分扣减时用行锁） |
| 唯一逻辑键 | `UNIQUE` 约束（如 `(tenant_id, provider_name)`） |
| Redis 辅助 | 速率限制用 ZSET 滑动窗口 |

### 7.4 文件路径速查

| 模块 | 路径 |
|------|------|
| 所有 Model | `api/models/model.py` |
| 用户/权限 Model | `api/models/account.py` |
| Workflow Model | `api/models/workflow.py` |
| 自定义字段类型 | `api/models/types.py` |
| 枚举定义 | `api/models/enums.py` |
| 数据库引擎配置 | `api/models/engine.py` |
| 对话服务 | `api/services/conversation_service.py` |
| 消息服务 | `api/services/message_service.py` |
| 文件服务 | `api/services/file_service.py` |
| 计费服务 | `api/services/billing_service.py` |
| 配额服务 | `api/services/quota_service.py` |
| 积分池服务 | `api/services/credit_pool_service.py` |
| 对话记忆恢复 | `api/core/memory/token_buffer_memory.py` |
| 文件访问控制 | `api/core/app/file_access/` |
| 权限装饰器 | `api/controllers/console/wraps.py` |
| RBAC 权限定义 | `api/core/rbac/entities.py` |
| 存储后端 | `api/extensions/storage/` |
| 文件常量（扩展名白名单） | `api/constants/__init__.py` |
| 文件大小/时间限制配置 | `api/configs/feature/__init__.py` |
| 应用模型类型 | `api/core/app/apps/message_based_app_generator.py` |
| 任务流水线 | `api/core/app/task_pipeline/easy_ui_based_generate_task_pipeline.py` |
| 删除任务 | `api/tasks/delete_conversation_task.py` |
| 消息事件信号 | `api/events/message_event.py` |
| 文件引用系统 | `api/core/workflow/file_reference.py` |
| 文件工厂 | `api/factories/file_factory/` |

---

> 本文件由 Dify 源码分析自动生成，可作为搭建类似平台的数据库架构参考。
