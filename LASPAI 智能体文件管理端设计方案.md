# LASPAI 智能体文件管理端设计方案

本文档描述 LASPAI 项目中文件管理服务器（File Server）的完整架构设计。文件服务器定位为 **数据服务**，统一管理物理文件存储、file_records 元数据以及 artifact_states 化学制品的全生命周期。客户端使用 SQLAlchemy ORM 连接数据库。

---

## 一、项目整体架构

文件管理端是 LASPAI 三层架构中的数据层，负责文件物理存储及所有相关元数据管理。智能体端内部直连访问；第三方客户通过 MCP 服务端代理。

```mermaid
flowchart TB
    AGENT["智能体端 (lasp_agent)"]
    THIRD["第三方客户"]

    AGENT -->|"直连（内部通路）"| FS["文件管理服务器<br/>(数据服务)"]
    AGENT -->|"MCP 协议（计算调用）"| MCP["MCP 服务端<br/>(计算网关)"]
    THIRD -->|"MCP 协议 / REST（资源接口）"| MCP

    MCP -->|"代理（第三方通路）"| FS

    FS -->|"读写"| STORAGE["物理存储 (files/)"]
    FS -->|"管理"| FILEDB["文件元数据库 (file_records)"]
    FS -->|"管理"| AGENTDB["Agent DB<br/>(artifact_states)"]
```

**职责边界**：

| 组件 | 职责 | 不负责 |
|------|------|--------|
| 文件管理服务器（数据服务） | 物理存储、file_records、artifact_states 管理、去重、文件生命周期 | 化学计算调度 |
| MCP 服务端（计算网关） | 调度计算集群、tool call 路由；对外代理文件请求（第三方通路） | 物理存储、元数据管理 |

**双通路说明**：

```text
内部通路（Agent 直连）：
  lasp_agent ──直连──▶ 文件管理服务器     ← 上传/下载/查询一步完成，低延迟

外部通路（第三方经 MCP）：
  第三方 ──MCP/REST──▶ MCP 服务端 ──代理──▶ 文件管理服务器   ← 安全受控，统一鉴权
```

- 智能体端直连文件服务器完成所有数据操作（物理文件 + artifact_states）
- 第三方客户只能通过 MCP 服务端间接访问

### 1.1 目录结构

```text
lasp_file_server/
├── core/
│   ├── database.py                # 数据库引擎（管理全部业务表）
│   ├── models.py                  # ORM 模型（file_records / artifact_states 等）
│   └── storage.py                 # 物理存储抽象（file_path 等）
├── services/
│   └── file_service.py            # upload_and_register / download_by_string_id
├── main.py
├── requirements.txt
└── .env
```

---

## 二、物理存储设计

### 2.1 目录结构

采用 **SHA256 内容寻址**，与 Git object store 同理。物理文件路径由内容哈希决定，天然去重，无需用户归属：

```text
files/
├── a3/
│   ├── a3b9c2d1e5f6789012345678901234567890abcd1234567890abcdef123456.pdb
│   └── a3f7e8a2c4d5e6789012345678901234567890abcd1234567890abcdef123456.cif
├── f8/
│   └── f83a9b2c4d5e6789012345678901234567890abcd1234567890abcdef123456.xyz
└── ...
```

| 设计要素 | 选择 | 理由 |
|---------|------|------|
| 寻址方式 | SHA256 前 2 字符 → 子目录 | 256 个桶均匀分布，防单目录膨胀 |
| 文件命名 | `{sha256}.{ext}` | 内容决定路径，多用户自动共享同一物理文件 |
| 用户隔离 | `file_records.user_id` 在数据库层 | 物理层不隔离，用户在 DB 层通过 record 归属区分 |

### 2.2 路径生成

```python
import os

def file_path(sha256: str, extension: str) -> str:
    """由 SHA256 计算物理路径：files/{前2位}/{完整sha256}.{ext}"""
    prefix = sha256[:2]
    path = os.path.join("files", prefix)
    os.makedirs(path, exist_ok=True)
    return os.path.join(path, f"{sha256}.{extension}")
```

如果未来迁移到对象存储（MinIO / S3），只需替换此函数的实现，数据库 schema 无需变动。

---

## 三、数据库设计

### 3.1 file_records — 文件元数据

`artifact_states` 与 `file_records` 为 M:N 关系。物理文件与 `file_records` 并非一一对应：多个 `file_records` 可指向同一物理文件（SHA256 相同），实现物理层去重而不依赖引用计数。

```sql
CREATE TABLE file_records (
    id              VARCHAR(36) PRIMARY KEY,
    user_id         VARCHAR(36) NOT NULL,               -- 多用户隔离
    filename        VARCHAR(255) NOT NULL,              -- 物理文件名 {sha256}.{ext}
    original_name   VARCHAR(255),                       -- 用户上传时的原始文件名
    extension       VARCHAR(16),                        -- pdb / cif / xyz / mol / sdf
    size_bytes      INT,                                -- 文件大小
    sha256          VARCHAR(64),                        -- SHA256 哈希（去重 + 完整性校验）
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_fr_user_id (user_id),
    INDEX idx_fr_sha256 (sha256)
);
```

| 字段 | 说明 |
|------|------|
| `id` | 主键，值存储在 `artifact_states.file_server_ids` JSON 数组中 |
| `user_id` | 多租户隔离，与 Agent 侧一致 |
| `filename` | 物理文件名 `{sha256}.{ext}`，同内容文件天然共享同一物理路径 |
| `original_name` | 用户上传时的原始文件名，供展示和下载还原 |
| `sha256` | 去重键：多 record 可共享同一物理文件；删除时按哈希计数决定是否清理物理文件 |

### 3.2 与 artifact_states 的关系（M:N）

```text
artifact_states                              file_records
──────────────                               ────────────
  string_id                                   id (PK)
  file_server_ids  ────[ "fs_a", ──────▶     filename
                         "fs_b" ] ────▶      original_name
  type                                        size_bytes
  chemical_metadata (JSON)                    sha256
  parent_artifact_id                          created_at
```

- `artifact_states.file_server_ids` 是 JSON 数组，存储一个或多个 `file_records.id`
- 仅需从 artifact 查文件，无反向查询需求，因此反范式设计可行
- 读取链路：`string_id → file_server_ids → 逐个查 file_records → 物理文件`

---

## 四、核心模块设计

### 4.1 文件上传（SHA256 去重，文件与 record 分离）

一次调用完成物理存储、去重检查、`file_records` 和 `artifact_states` 写入，返回 `string_id`。物理层通过 SHA256 去重：多 record 指向同一物理文件，但 record 各自独立。

```python
import hashlib

class FileService:
    def upload_and_register(
        self, user_id: str, content: bytes,
        original_name: str, artifact_type: str,
        chemical_metadata: dict = None,
        parent_artifact_id: str = None,
    ) -> str:
        sha256 = hashlib.sha256(content).hexdigest()
        ext = original_name.rsplit(".", 1)[-1] if "." in original_name else ""
        file_id = generate_uuid()

        # 物理层去重：相同 SHA256 的文件不重复写入
        physical_filename = f"{sha256}.{ext}"
        path = file_path(sha256, ext)
        if not os.path.exists(path):
            with open(path, "wb") as f:
                f.write(content)

        # 始终创建新的 file_record
        record = FileRecord(
            id=file_id,
            user_id=user_id,
            filename=physical_filename,
            original_name=original_name,
            extension=ext,
            size_bytes=len(content),
            sha256=sha256,
        )
        db.add(record)

        # 创建 artifact_states
        string_id = f"{artifact_type[:3]}_{generate_short_id()}"
        artifact = ArtifactState(
            string_id=string_id,
            file_server_ids=[file_id],
            type=artifact_type,
            user_id=user_id,
            conversation_id=conversation_id,
            chemical_metadata=chemical_metadata or {},
            parent_artifact_id=parent_artifact_id,
            source="upload",
        )
        db.add(artifact)
        db.commit()
        return string_id
```

### 4.2 文件下载

按 `string_id` 一步完成查询和读取：

```python
    def download_by_string_id(self, string_id: str) -> bytes:
        artifact = db.query(ArtifactState).filter_by(string_id=string_id).first()
        if not artifact or not artifact.file_server_ids:
            raise FileNotFoundError(f"Artifact {string_id} not found")

        file_record = db.query(FileRecord).filter_by(
            id=artifact.file_server_ids[0]
        ).first()

        path = os.path.join("files", file_record.filename[:2],
                            file_record.filename)
        with open(path, "rb") as f:
            return f.read()
```

### 4.3 文件删除

删除 `file_record` 前检查是否仍有其他 record 指向同一物理文件，若无则一并删除物理文件。数据库操作在事务内完成，物理文件删除为 best-effort（失败由定期清理接管）：

```python
    def delete_record(self, record_id: str):
        with db.transaction():
            record = db.query(FileRecord).filter_by(id=record_id).first()
            if not record:
                return
            db.delete(record)
            remaining = db.query(FileRecord).filter_by(sha256=record.sha256).count()

        # 物理文件删除在事务外（文件系统无法回滚）
        if remaining == 0:
            path = os.path.join("files", record.filename[:2], record.filename)
            if os.path.exists(path):
                os.remove(path)
```

### 4.4 定期清理

定时任务（如每日）扫描 `artifact_states` 和 `conversations`，清理不再需要的物理文件：

```python
    def cleanup_orphan_files(self):
        """清理所属对话已删除或归档超期的物理文件。"""
        cutoff_deleted = datetime.utcnow() - timedelta(days=30)   # 软删除宽限期
        cutoff_archived = datetime.utcnow() - timedelta(days=7)   # 归档保留期

        orphan_artifacts = db.query(ArtifactState).join(Conversation).filter(
            or_(
                Conversation.deleted_at < cutoff_deleted,
                and_(Conversation.status == 'archived',
                     Conversation.updated_at < cutoff_archived),
            )
        ).all()

        for artifact in orphan_artifacts:
            for file_id in (artifact.file_server_ids or []):
                record = db.query(FileRecord).filter_by(id=file_id).first()
                if not record:
                    continue
                db.delete(record)
                # 物理文件只在无其他 record 引用时删除
                if db.query(FileRecord).filter_by(sha256=record.sha256).count() == 0:
                    path = os.path.join("files", record.filename[:2], record.filename)
                    if os.path.exists(path):
                        os.remove(path)
        db.commit()
```

> 清理依据为 conversation 状态，而非文件访问时间。物理文件删除后 artifact_states 保留，`file_server_ids` 中对应 ID 标记为失效。

---

## 五、典型数据流

### 5.1 用户上传化学结构文件

```mermaid
sequenceDiagram
    participant FE as 前端
    participant AG as 智能体端
    participant FS as 文件服务器

    FE->>AG: POST /chat (含文件)
    AG->>FS: upload_and_register(user_id, content, name, type, metadata)
    FS->>FS: 计算 SHA256
    alt 已存在（SHA256 命中）
        FS->>FS: 复用已有物理文件，创建新 file_record
    else 新文件
        FS->>FS: 写入 files/{sha256[:2]}/{sha256}.{ext}
        FS->>FS: 写入 file_record
    end
    FS->>FS: 写入 artifact_states (file_server_ids=[...])
    FS-->>AG: 返回 string_id
    AG-->>FE: SSE 推送上传完成
```

### 5.2 MCP 工具调用中读取文件

```mermaid
sequenceDiagram
    participant SA as 子智能体
    participant FS as 文件服务器

    SA->>FS: download_by_string_id(string_id)
    FS->>FS: 查 artifact_states → 获取 file_server_ids
    FS->>FS: 查 file_records → 定位物理文件
    FS-->>SA: 返回 PDB/CIF 文本内容
```

---

## 六、关键设计决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 物理存储 | 本地文件系统单目录平铺 + 次级目录 | 小型服务够用，零运维成本；子目录零成本防单目录膨胀 |
| 次级目录 | `user_id` 次级目录 | 用户 ID 不变、合法文件名；删除用户 = 删除目录 |
| 文件命名 | `{file_id}.{ext}` | UUID + 扩展名保证唯一且可读 |
| 去重 | SHA256（不依赖引用计数） | 多 record 共享同一物理文件；删除时按 SHA256 计数判断是否清理物理文件 |
| 元数据分离 | `file_records` ≠ `artifact_states` | 存储信息与化学语义解耦，各自独立演进 |
| 预留升级空间 | 预留 `file_path()` 抽象 | 日后迁移 MinIO/S3 只需改这一行实现 |
| 不启用对象存储 | 暂不引入 MinIO/S3 | 当前规模不需要，过度设计徒增运维复杂度 |
| 大文件 | 不做分片 | 化学结构文件（PDB/CIF）通常 < 500KB，单次读写即可 |
| 生命周期 | 文件服务器统一管理 | 物理文件 + artifact_states 同生命周期，删除时一并清理 |
| 文件清理 | 基于 conversation 状态定期清理 | 对话删除 30 天或归档 7 天后清理物理文件；不使用引用计数 |
| Artifact ↔ File | M:N 关系 | 一个 artifact 可含多个文件；去重时多 artifact 共享同一文件 |
| 反范式 | `file_server_ids` JSON 数组 | 仅 artifact→file 方向查询，无需反向，JSON 数组更简单 |
| 一致性 | DB 事务 + 物理文件 best-effort | DB 层原子提交；物理文件失败由定期清理最终一致 |

---

## 七、与其他模块的协作协议

### 7.1 智能体端 ↔ 文件服务器（内部直连，统一入口）

智能体端所有数据操作通过文件服务器一步完成，不再跨服务拼接：

```text
智能体端                                              文件服务器
───────                                              ──────────
upload_and_register(user_id, content, ...)  ──────▶ 写物理文件 + file_records + artifact_states
                                                        → 返回 string_id
download_by_string_id(string_id)            ──────▶ 查 artifact_states → 查 file_records
                                                        → 读物理文件 → 返回 bytes
query_artifacts(user_id, type)              ──────▶ 查询 artifact_states → 返回列表
```

### 7.2 MCP 服务端 ↔ 文件服务器（代理 & 计算后写入）

第三方不直连文件服务器，通过 MCP 代理；MCP 自身计算产出也通过此通路写入：

```text
MCP 服务端                                              文件服务器
──────────                                              ──────────
代理上传（第三方请求）        ─────────────────────▶ upload_and_register(...) → 返回 string_id
代理下载（第三方请求）        ─────────────────────▶ download_by_string_id(...) → 返回 bytes
计算完成后写入               ─────────────────────▶ upload_and_register(..., source="computation")
```

### 7.3 两表关系映射

```text
artifact_states                                 file_records
──────────────                                  ────────────
file_server_id (FK)        ─────────────────▶  id (PK)
string_id                                      filename
chemical_metadata                               original_name
type                                           size_bytes
source                                         sha256
```

两表均由文件服务器管理，通过 `file_server_id` 关联。查询链路：`string_id → artifact_states → file_server_id → file_records → 物理文件`。

---

MCP 工具计算产出新文件时，调用 `upload()` 获取 `file_server_id`，再写入 `artifact_states` 记录。

---
