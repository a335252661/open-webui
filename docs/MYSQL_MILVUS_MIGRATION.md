# Open WebUI 企业级存储迁移方案：SQLite + Chroma → MySQL + Milvus

> 适用范围：本项目（Open WebUI v0.11.0，后端 FastAPI / SQLAlchemy 2.0.50 async）
> 目标：把关系数据从本地 SQLite（`backend/data/webui.db`）迁移到 MySQL，把向量数据从本地 Chroma（`backend/data/vector_db`）迁移到 Milvus，支撑企业级多实例、高并发、HA 部署。

---

## 1. 现状与目标

### 1.1 当前架构（单机嵌入式）

| 数据 | 存储 | 位置 | 形态 |
|---|---|---|---|
| 关系数据（用户/会话/消息/模型/知识库元数据/配置） | SQLite | `backend/data/webui.db` | 单文件 |
| 向量数据（RAG 分块、agent 记忆） | Chroma (PersistentClient) | `backend/data/vector_db/` | 本地文件/SQLite |
| 文件附件 | 本地文件系统 | `backend/data/uploads/` | 文件 |

### 1.2 目标架构（独立服务）

| 数据 | 存储 | 接入方式 |
|---|---|---|
| 关系数据 | MySQL 8.x | `DATABASE_URL=mysql+pymysql://...`（运行期走 asyncmy 异步驱动） |
| 向量数据 | Milvus Standalone | `VECTOR_DB=milvus` + `MILVUS_URI=http://<host>:19530` |
| 文件附件 | 保持本地磁盘（可后续迁移对象存储 MinIO/S3） | 不变 |

迁移后：多后端实例可共享 MySQL + Milvus，具备独立运维、备份、扩容能力。

---

## 2. 可行性结论（重要，先读）

| 目标 | 官方支持程度 | 需要的改动 |
|---|---|---|
| **Milvus** | ✅ 官方内置（`pymilvus==2.6.14` 已在 requirements） | 仅配置环境变量，无需改代码 |
| **MySQL** | ⚠️ **本版本主库未官方支持** | 需 ① 安装异步驱动 `asyncmy`；② 改一行代码（见 §4.1） |

**原因（代码实证）：**

1. 运行期全部走**异步引擎**（[db.py](file:///d:/project/private/open-webui/backend/open_webui/internal/db.py#L360-L362) 的 `create_async_engine`）。
2. 项目 requirements 只有 `PyMySQL==1.2.0`（同步驱动），实测 `mysql+pymysql://` 传入异步引擎直接报错：

   ```
   sqlalchemy.exc.InvalidRequestError: The asyncio extension requires an async driver
   to be used. The loaded 'pymysql' is not async.
   ```

3. URL 转换函数 [\_make_async_url](file:///d:/project/private/open-webui/backend/open_webui/internal/db.py#L204-L229) 只处理了 sqlite / postgresql，对 mysql 是"原样返回"（注释明确写 `For other dialects, return as-is`）。

因此：**纯配置切 MySQL 不可行，必须安装 asyncmy 并给 `_make_async_url` 加一行转换**（与 PostgreSQL 用 psycopg 双模驱动的思路一致：同步引擎/Alembic 用 pymysql，异步引擎用 asyncmy）。

---

## 3. 迁移前置条件

1. 准备 MySQL 8.x（独立实例），建议 UTF-8：`CREATE DATABASE openwebui CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`
2. 准备 Milvus Standalone（部署见 §5）。
3. 备份当前数据（迁移前必须）：
   ```powershell
   # 停止后端后执行
   Copy-Item backend\data\webui.db backend\data\webui.db.bak
   Copy-Item -Recurse backend\data\vector_db backend\data\vector_db.bak
   ```
4. 保持 embedding 模型一致（迁移前后必须用同一个 embedding 模型，否则向量维度/语义不一致）：
   默认模型 `sentence-transformers/all-MiniLM-L6-v2`（384 维，[config.py](file:///d:/project/private/open-webui/backend/open_webui/config.py#L990-L990)），也可显式设置 `RAG_EMBEDDING_MODEL` 固定版本。

---

## 4. 代码与依赖调整（仅 MySQL 需要）

### 4.1 安装异步驱动

```powershell
cd backend
.venv\Scripts\pip install asyncmy
```

（若装 asyncmy 遇到编译问题，可改用纯 Python 的 `aiomysql`，下述代码改动同样适用。）

建议同时写入 `backend/requirements.txt`（与官方依赖保持一致）：

```
asyncmy==0.2.10
```

### 4.2 修改 `_make_async_url`（一行）

文件：[backend/open_webui/internal/db.py](file:///d:/project/private/open-webui/backend/open_webui/internal/db.py#L204-L229)

在 postgres 分支之后、`return url` 之前增加：

```python
    # pymysql (sync/alembic) → asyncmy (async runtime), mirrors the psycopg dual-mode pattern
    if url.startswith('mysql+pymysql://'):
        return url.replace('mysql+pymysql://', 'mysql+asyncmy://', 1)
```

改完后同步引擎、Alembic 继续用 `mysql+pymysql://`，异步引擎自动切到 `mysql+asyncmy://`。

---

## 5. 部署 Milvus（Docker Compose，Standalone）

创建 `deploy/milvus/docker-compose.yml`（标准三件套：etcd + minio + milvus）：

```yaml
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.18
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
    volumes:
      - etcd:/etcd
    command: etcd -advertise-client-urls=http://etcd:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd

  minio:
    image: minio/minio:RELEASE.2025-04-22T22-12-26Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - minio:/minio_data
    command: minio server /minio_data --console-address ":9001"

  milvus:
    image: milvusdb/milvus:v2.5.9
    command: ["milvus", "run", "standalone"]
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
    ports:
      - "19530:19530"
      - "9091:9091"
    volumes:
      - milvus:/var/lib/milvus
    depends_on:
      - etcd
      - minio

volumes:
  etcd:
  minio:
  milvus:
```

启动：

```powershell
docker compose -f deploy\milvus\docker-compose.yml up -d
```

验证：

```powershell
docker compose -f deploy\milvus\docker-compose.yml ps
# 确认 milvus 容器 healthy；19530 端口监听
```

---

## 6. 配置后端环境变量

在启动后端的环境（或 `.env`）中设置：

```env
# ── 关系库：MySQL ──
DATABASE_URL=mysql+pymysql://openwebui:你的密码@127.0.0.1:3306/openwebui?charset=utf8mb4
# 连接池（建议多实例时调大）
DATABASE_POOL_SIZE=20
DATABASE_POOL_MAX_OVERFLOW=20

# ── 向量库：Milvus ──
VECTOR_DB=milvus
MILVUS_URI=http://127.0.0.1:19530
# MILVUS_TOKEN=xxx            # 若开启鉴权
# MILVUS_INDEX_TYPE=HNSW      # 默认即可
# MILVUS_METRIC_TYPE=COSINE   # 默认即可

# ── 其它保持原值（WEBUI_SECRET_KEY、CORS_ALLOW_ORIGIN 等）──
```

> 注意：不要用 `DATABASE_TYPE=mysql` 拼 URL —— 它会产生无驱动的 `mysql://user:pass@host:port/db`，而 SQLAlchemy 默认 mysql 方言是 MySQLdb（未安装）。**必须显式写 `mysql+pymysql://`**（见 [env.py](file:///d:/project/private/open-webui/backend/open_webui/env.py#L267-L296)）。

---

## 7. 建表：让 Alembic 在 MySQL 生成全量 schema

Open WebUI 启动时（`ENABLE_DB_MIGRATIONS` 默认开启）会自动执行 `alembic upgrade head`（[config.py](file:///d:/project/private/open-webui/backend/open_webui/config.py#L62-L79)），在 MySQL 上创建全部表（`init` → 各增量版本，见 [migrations/versions](file:///d:/project/private/open-webui/backend/open_webui/migrations/versions)）。

也可手动触发：

```powershell
cd backend
.venv\Scripts\python -m alembic -c open_webui\alembic.ini upgrade head
```

> Alembic 走的是**同步引擎**（pymysql），与本方案第 4 步的改动不冲突。

---

## 8. 关系数据迁移：SQLite → MySQL

由于 Open WebUI 没有内置的 SQLite→MySQL 迁移命令，用一个脚本按表整体搬运。列类型兼容（主键为 TEXT，JSON 以 TEXT 存储，见 [JSONField](file:///d:/project/private/open-webui/backend/open_webui/internal/db.py#L123-L148)），可直接逐表复制。

脚本（一次性脚本，执行后删除）：

```python
# scripts/migrate_sqlite_to_mysql.py
"""SQLite -> MySQL 数据迁移脚本。先建表（alembic），再执行本脚本。"""
from sqlalchemy import create_engine, MetaData, select

SRC = 'sqlite:///d:/project/private/open-webui/backend/data/webui.db'
DST = 'mysql+pymysql://openwebui:你的密码@127.0.0.1:3306/openwebui?charset=utf8mb4'

src = create_engine(SRC)
dst = create_engine(DST)
meta = MetaData()
meta.reflect(bind=src)

with dst.begin() as conn:
    for table in meta.sorted_tables:
        rows = src.execute(select(table)).mappings().all()
        if rows:
            conn.execute(table.insert(), [dict(r) for r in rows])
        print(f'{table.name}: {len(rows)}')
print('done')
```

执行：

```powershell
cd backend
.venv\Scripts\python scripts\migrate_sqlite_to_mysql.py
```

校验（两边行数一致）：

```sql
-- MySQL
SELECT (SELECT COUNT(*) FROM user)  AS users,
       (SELECT COUNT(*) FROM chat)  AS chats,
       (SELECT COUNT(*) FROM memory) AS memories;
```

> **大 JSON 字段风险**：MySQL `TEXT` 上限 64KB。`chat.data`、`chat_message.data/content` 等 JSON 列在极端情况下可能超限被截断。若库里有超大会话，迁移前先把这些列升级为 `LONGTEXT`：
> ```sql
> ALTER TABLE chat MODIFY COLUMN data LONGTEXT;
> ALTER TABLE chat_message MODIFY COLUMN data LONGTEXT;
> ```

---

## 9. 向量数据迁移：Chroma → Milvus

Milvus 的 collection 向量维度在创建时固定，必须与 embedding 模型一致（见 §3.4）。两种方式任选：

### 方式 A（推荐）：切换后重新入库

1. 停后端 → 按 §6 设置 `VECTOR_DB=milvus` → 启后端。
2. 在管理端对每个知识库**重新上传/重新入库**（docs 入库会重新切分 + embedding，语义等价）。
3. agent 的 `user-memory-*` 记忆集合随新会话重新生成。

优点：无需处理 collection 结构差异，最稳妥；缺点：需要人工/脚本触发重新入库。

### 方式 B：脚本直拷（批量场景）

用 Open WebUI 的 `VECTOR_DB_CLIENT` 统一接口（[factory.py](file:///d:/project/private/open-webui/backend/open_webui/retrieval/vector/factory.py)）从旧 Chroma 读、写入新 Milvus：

```python
# scripts/migrate_chroma_to_milvus.py（在 VECTOR_DB=chroma 的旧环境读，VECTOR_DB=milvus 的新环境写）
import os
os.environ['VECTOR_DB'] = 'milvus'   # 写端
from open_webui.retrieval.vector.factory import VECTOR_DB_CLIENT as dst
os.environ['VECTOR_DB'] = 'chroma'   # 读端
from open_webui.retrieval.vector.factory import VECTOR_DB_CLIENT as src

for col in ['knowledge-<id>', 'user-memory-<uid>', ...]:  # 枚举实际 collection
    items = src.get(col)                      # 取全部向量+元数据
    dst.insert(col, items)                    # 写入 Milvus
    print(col, len(items))
```

> 注意：两个 client 的 `import open_webui.config` 是进程级单例，同进程内切 `VECTOR_DB` 不可靠。**建议分两步**：先在旧环境导出（dump 到文件），再在新环境导入。若 collection 数量少，优先用方式 A。

---

## 10. 启动与验证

```powershell
# 后端
cd backend
.venv\Scripts\uvicorn open_webui.main:app --host 0.0.0.0 --port 8080
```

验证清单：

| 项 | 方法 | 预期 |
|---|---|---|
| 关系库连通 | `http://localhost:8080/health` | 返回正常，无 DB 报错 |
| MySQL 落库 | MySQL 中查 `user`/`chat` 表 | 有迁移过来的数据 |
| Milvus 连通 | 管理端"文档"新建知识库入库 | 成功后 Milvus 出现新 collection |
| RAG 检索 | 对话中 @知识库 提问 | 能命中内容 |
| agent 记忆 | agent 会话中使用记忆工具 | `user-memory-*` 写入 Milvus |
| 并发 | 起 2 个后端实例指向同一 MySQL/Milvus | 均正常读写 |

---

## 11. 回滚方案

1. 停后端。
2. 还原环境变量（去掉 `DATABASE_URL`/`VECTOR_DB`，或指回本地）。
3. 用备份还原本地数据：
   ```powershell
   Copy-Item backend\data\webui.db.bak backend\data\webui.db
   Remove-Item -Recurse backend\data\vector_db
   Copy-Item -Recurse backend\data\vector_db.bak backend\data\vector_db
   ```
4. 重启后端。

> 迁移后新产生的数据在 MySQL/Milvus，若需回滚会丢失该窗口期数据 —— 迁移建议安排在维护窗口，回滚窗口内尽量只读。

---

## 12. 风险与注意事项汇总

| 风险 | 说明 | 缓解 |
|---|---|---|
| MySQL 非官方主库 | 本版本未官方支持，升级 Open WebUI 需重新验证 | 保留 `_make_async_url` 补丁；升级前回归测试 |
| asyncmy 编译 | Windows 下偶有编译问题 | 可换 `aiomysql`（纯 Python） |
| TEXT 64KB 截断 | 超大 JSON 会话可能超限 | 迁移前将大列 ALTER 为 LONGTEXT |
| 向量维度不一致 | 换 embedding 模型会导致新旧向量语义/维度不匹配 | 固定同一 `RAG_EMBEDDING_MODEL` |
| Chroma→Milvus 无官方迁移 | 需重新入库或脚本直拷 | 优先"切换后重新入库" |
| 保留字/字符集 | 表名/字符串编码差异 | MySQL 用 utf8mb4；SQLAlchemy 自动加反引号 |
| 升级兼容 | 后续版本迁移脚本可能变更 schema | 升级前备份，按版本发布说明处理 |

---

## 13. 后续可选增强（企业级延伸）

- 文件附件迁移到对象存储（MinIO/S3），`STORAGE_PROVIDER` 相关配置。
- 加 Redis 支撑任务队列与实时消息（`REDIS_URL`）。
- MySQL 主从、Milvus 集群 + 多副本，滚动升级与自动备份。
