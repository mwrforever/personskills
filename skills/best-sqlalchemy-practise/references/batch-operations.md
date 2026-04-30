# 七、批量操作性能调优

## 7.1 禁止在循环中 `session.add()`

### ✅ 强制规范

```python
# 方式一：使用 add_all + 分批
async def bulk_insert_users(users_data: list[dict], batch_size: int = 1000):
    async with AsyncSessionFactory() as session:
        for i in range(0, len(users_data), batch_size):
            batch = users_data[i:i + batch_size]
            session.add_all([User(**data) for data in batch])
            await session.flush()  # 每 1000 条 flush 一次，释放内存
        await session.commit()
```

### ❌ 严禁操作

```python
# 致命错误：循环中 session.add()
for data in users_data:
    user = User(**data)
    session.add(user)  # 每次 add 都触发一次 INSERT 的准备工作
# 10000 条数据 = 10000 次 Python ↔ 数据库交互
# 性能比 add_all 慢 10-50 倍
# identity_map 中积累 10000 个对象，内存占用巨大
```

**底层原理解释**：每次 `session.add()` 触发 ORM 的事件系统：`before_insert`、`before_flush` 等钩子被调用，对象被加入 identity_map，关联关系被级联处理。`add_all()` 将所有这些开销分摊到一次操作中。更重要的是，每次 `flush()` 将一批对象的 INSERT 打包成一条网络消息发送——减少 TCP 往返次数是数据库性能优化的核心手段。

---

## 7.2 ORM 与 Core 的性能分水岭

### ✅ 强制规范

```python
from sqlalchemy import insert

# 万级数据（< 10000 行）— ORM add_all 足够
session.add_all([User(**data) for data in small_batch])
await session.commit()

# 十万级数据（10000 - 100000 行）— ORM + 分批 flush
for batch in chunks(data, 5000):
    session.add_all([User(**d) for d in batch])
    await session.flush()
await session.commit()

# 百万级数据（> 100000 行）— 必须降级到 Core
async with engine.begin() as conn:
    for batch in chunks(data, 10000):
        await conn.execute(
            insert(User).values(batch)  # ← 绕过 ORM，直接 Core INSERT
        )
    # engine.begin() 自动 commit

# 极度场景（> 100万行）— 使用 PostgreSQL COPY 协议
async with engine.begin() as conn:
    raw_conn = await conn.get_raw_connection()
    # 使用 asyncpg 的 copy_records_to_table
    await raw_conn.copy_records_to_table(
        "user", records=data, columns=["name", "email", ...]
    )
```

### ❌ 严禁操作

```python
# 致命错误：百万级数据用 ORM add_all
session.add_all([User(**d) for d in million_rows])
await session.commit()
# 问题：(1) identity_map OOM — 百万个对象在 Python 堆中
#       (2) ORM 为每个对象生成 INSERT — 百万条独立的 SQL 语句
#       (3) 无法利用数据库的批量加载优化（如 PostgreSQL 的 COPY）
#       (4) commit 超时 — 数百万行的单个事务可能触发 WAL 溢出
```

**底层原理解释**：ORM 的核心价值在于将数据库行映射为 Python 对象，并追踪它们的变更。当你要插入一百万行新数据时，你不需要"追踪变更"——这些数据只会写入一次。ORM 的 identity_map、事件系统、级联处理此时都是纯粹的开销。直接使用 Core 的 `insert().values()` 绕过这些开销，可以比 ORM 快 10-50 倍。

---

## 7.3 批量 UPDATE/DELETE 也必须绕过 ORM

```python
from sqlalchemy import update, delete

# ✅ 正确：使用 Core 进行批量操作
stmt = (
    update(User)
    .where(User.department_id == old_dept_id)
    .values(department_id=new_dept_id)
)
await session.execute(stmt)  # 一条 SQL，更新 50000 行

# ❌ 错误：用 ORM 逐行更新
users = await session.execute(select(User).where(User.department_id == old_dept_id))
for user in users.scalars():
    user.department_id = new_dept_id  # 50000 个 ORM 对象被加载到内存
# 50000 次 Python 属性赋值 + 50000 次 flush 检查
```
