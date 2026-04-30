# 二、引擎与连接池层

## 2.1 连接池参数的数学逻辑

### ✅ 强制规范

```python
from sqlalchemy.ext.asyncio import create_async_engine

# 生产级连接池计算
DB_MAX_CONNECTIONS = 200       # PostgreSQL 默认 max_connections
WORKER_COUNT = 4               # gunicorn/uvicorn worker 数量
POOL_SIZE = 10                 # 每个进程的持久连接数
MAX_OVERFLOW = 20              # 峰值时额外可创建的连接数

# 数学约束：
# (POOL_SIZE + MAX_OVERFLOW) × WORKER_COUNT ≤ DB_MAX_CONNECTIONS × 0.8
# (10 + 20) × 4 = 120 ≤ 160 ✅ — 为管理员操作和监控预留 40 个连接

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@host:5432/db",
    pool_size=POOL_SIZE,
    max_overflow=MAX_OVERFLOW,
    pool_timeout=30,           # 等待可用连接的最大秒数，超时抛出 TimeoutError
    pool_recycle=3600,         # 连接最大存活秒数，防中间件（如 PgBouncer）断连
    pool_pre_ping=True,        # 每次 checkout 前发送 SELECT 1 验证连接有效性
    echo=False,                # 生产环境绝对禁止 True，会泄漏敏感数据到日志
)
```

### ❌ 严禁操作

```python
# 致命配置：使用默认值
engine = create_engine("postgresql://...")
# 默认 pool_size=5, max_overflow=10, pool_timeout=30
# 8 个 worker × 5 = 40 个持久连接看起来没问题
# 但一旦 3-4 个 worker 同时处理慢查询，max_overflow 被用满后，
# 新请求直接爆 TimeoutError，用户看到的是 502/504 而不是你的应用正常报错

# 致命配置：把 pool_recycle 设得太短
engine = create_engine("...", pool_recycle=60)
# 每分钟强制断开所有连接重建，连接风暴直接打挂数据库
```

**底层原理解释**：
- `pool_size`：持久连接在 pool 中保持打开，避免频繁 TCP 三次握手和 PostgreSQL 认证开销。每个持久连接在 PostgreSQL 端占用约 2-4MB 内存。
- `max_overflow`：当 pool 中的连接全部被占用时，允许额外创建的最多连接数。这些连接在用完后立即关闭，不会归还 pool。溢出连接是你的"safety valve"——应该有但不应完全依赖。
- `pool_timeout`：不是连接的最长存活时间（那是 `pool_recycle`），而是当 pool 已满（pool_size + max_overflow 全部占用）时，新请求等待可用连接的最长时间。
- `pool_recycle`：针对连接池中间件（pgbouncer）或防火墙超时的防御措施。设为 3600（1 小时）是一个安全的起点。
- **总连接数公式**：`(pool_size + max_overflow) × worker_count + (2-5 admin connections)` 必须小于 `PG max_connections`。

---

## 2.2 `pool_pre_ping=True` — 非可选配置

```python
# 绝对必须 — 没有例外
engine = create_engine("...", pool_pre_ping=True)
```

**底层原理解释**：当 PostgreSQL 服务器重启、pgbouncer 断开长连接、或者网络设备静默丢弃 TCP 连接（socket timeout）时，连接池中的连接可以悄无声息地"死亡"。不开启 `pool_pre_ping`，你的应用会尝试在死连接上执行 SQL，收到 `OperationalError` 或 `InterfaceError`，然后该请求失败——而下一次同一个工作进程中的请求仍然可能分到这个死连接，导致级联失败。`pool_pre_ping` 每次从池中取出连接时先发送一条轻量级的 `SELECT 1` 验证连接有效性。死连接会被自动回收，对调用者完全透明。开销极小（<0.1ms），但在生产环境中这是生存必需品。

---

## 2.3 同步 vs 异步引擎：驱动选择

### ✅ 强制规范

```python
# 异步首选 — asyncpg（PostgreSQL 的最佳异步驱动）
from sqlalchemy.ext.asyncio import create_async_engine
async_engine = create_async_engine(
    "postgresql+asyncpg://user:pass@host/db",
    # asyncpg 使用 PostgreSQL 原生二进制协议，性能最优
    # 支持 prepared statements 和 LISTEN/NOTIFY
)

# 同步首选 — psycopg2（cpp 扩展，成熟稳定）
from sqlalchemy import create_engine
sync_engine = create_engine(
    "postgresql+psycopg2://user:pass@host/db",
)
```

### ❌ 严禁操作

```python
# 绝不在异步代码中使用同步引擎
async def my_handler():
    result = sync_engine.execute(...)  # 阻塞事件循环！整个 worker 挂起！

# 绝不在异步引擎中使用 psycopg2
engine = create_async_engine("postgresql+psycopg2://...")
# psycopg2 不支持 asyncio，SQLAlchemy 会用线程池包装它，性能大打折扣

# 对于 MySQL 用户：aiomysql 和 asyncmy 的选择
# aiomysql 是纯 Python 实现，性能差
# asyncmy 基于 Cython，性能接近 pymysql 的同步版本
```

**底层原理解释**：asyncpg 直接实现了 PostgreSQL 的二进制前端/后端协议（FE/BE Protocol），避免了 psycopg2 的 libpq C 库回调模型。在 asyncio 事件循环中，asyncpg 原生支持非阻塞 I/O，单连接吞吐量可比同步驱动高 2-5 倍。但 asyncpg 不直接兼容 psycopg2 的自定义类型适配器和某些 PostgreSQL 扩展——在需要这些功能的场景中，使用 psycopg2 + `sqlalchemy[asyncio]`（线程池模式）作为降级方案。
