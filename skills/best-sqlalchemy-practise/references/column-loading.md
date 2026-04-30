# 按需列加载与内存管控规范

**核心指导思想**：数据库不存在"只读一个小字段"的优化，`SELECT *` 永远会将整行数据（包括巨大的 TEXT/JSONB）加载到内存和网络带宽中。必须根据业务上下文，严格在三种按需策略中做唯一选择，杜绝资源浪费与隐式查询灾难。

---

## 策略一：纯数据投影 —— 返回轻量级 Row（最高频、最优解）

**适用场景**：列表页渲染、下拉框选项、导出报表、外部接口数据组装。**不需要**对实体进行属性修改，也不需要调用实体的实例方法。

**技术手段**：在 `select()` 中传入具体的列属性，而不是传入 Model 类。

**返回形态**：返回底层 C 实现的轻量级 `Row`（类似 namedtuple），而非重量级 ORM 实例。

### ✅ 正确示例

```python
# 同步写法
stmt = select(
    User.id,
    User.username
).where(User.is_active == True)

with SessionLocal() as session:
    rows = session.execute(stmt).all()
    result = [{"id": row.id, "username": row.username} for row in rows]

# 异步写法
stmt = select(Order.id, Order.status).where(Order.user_id == user_id)

async with async_session() as session:
    result = await session.execute(stmt)
    rows = result.all()  # 异步返回的也是 Row 序列
```

取值方式：支持 `row.column_name`、`row[0]` 索引、或 `id, name = row` 解构。

**⚠️ 绝对红线**：严禁在此场景下使用 `.scalars()`。`scalars()` 用于提取单列的第一行作为标量，用在多列查询中会直接导致数据丢失或类型错乱。

### ❌ 严禁操作

```python
# 禁止：假按需（查出全量重量级对象，然后在 Python 层面丢弃无用字段）
users = session.execute(select(User)).scalars().all()
result = [{"id": u.id, "username": u.username} for u in users]
# 灾难后果：巨大的 profile_json、bio_text 被无意义地加载、实例化，引发 OOM 和 GC 停顿。
```

**底层原理解释**：当 `select()` 中传入 Model 类（`select(User)`），SQLAlchemy 会发出 `SELECT *` 并创建完整的 ORM 实例——每个实例包含 identity_map 注册、属性追踪字典、脏检查基础设施等。对于一个 20 列、其中包含 50KB JSONB 列的表，查询 10000 行时：ORM 实例消耗约 500MB+，而 `Row` 对象仅消耗约 10-20MB。差距可达 25-50 倍。

---

## 策略二：ORM 实例部分加载 —— 返回残缺对象（高危、需严密防守）

**适用场景**：业务逻辑必须拿到一个 ORM 对象（例如需要调用对象的方法、需要传入其他接受 ORM 类型的函数、或者后续确实要修改该对象的个别字段并 commit），但该表存在极大的字段（如 `article_content`），且当前逻辑确定不会触碰这些大字段。

**技术手段**：使用 `options(load_only(...))`。

**⚠️ 致命陷阱：隐式 IO**

使用 `load_only` 后，未指定的字段不会被加载（值为 None 或未初始化状态）。如果后续代码不小心读取了未加载的字段，SQLAlchemy 会**静默发起一条额外的 SQL** 去数据库补全数据。这在循环中就是 N+1 灾难！

### 🛡️ 生产级防御方案：必须搭配 raiseload 拦截器

凡是使用 `load_only()`，**必须同时搭配 `raiseload('*')` 作为全局兜底！** 将隐式 IO 转化为显式崩溃。

### ✅ 正确示例

```python
from sqlalchemy.orm import load_only, raiseload

# 场景：仅更新用户的最后登录时间，绝对不需要加载用户的个人简介
stmt = (
    select(User)
    .options(
        load_only(User.id, User.username, User.last_login_time),
        raiseload('*')  # 铁律：除了 load_only 指定的列，访问其他任何属性直接抛错！
    )
    .where(User.id == 1)
)

with SessionLocal() as session:
    user = session.execute(stmt).scalar_one()
    user.last_login_time = func.now()
    session.commit()
    # 假设后面有人不小心写了这行：
    # print(user.bio)
    # 结果：直接抛出 sqlalchemy.exc.InvalidRequestError，在测试期立刻暴露问题！
```

### 补充语法：`defer()` 的等价写法

除了正向指定要加载的列，也可以反向指定要延迟加载的列（两者等价，`load_only` 在现代代码中语义更清晰）：

```python
# 等价于 load_only(User.id, User.username)
stmt = select(User).options(
    defer(User.bio),
    defer(User.profile_json),
    raiseload('*')
)
```

### ❌ 严禁操作

```python
# 致命错误：裸用 load_only，不设防
stmt = select(User).options(load_only(User.id, User.username))
user = session.execute(stmt).scalar_one()
send_notification_email(user.email)  # 灾难！email 未加载，静默触发全表回查
# 如果这段代码在一个循环中（遍历 1000 个用户发邮件），
# 会产生 1 + 1000 条 SQL！与 N+1 查询完全等价的灾难模式。
```

---

## 策略三：标量聚合查询 —— 提取单一计算值

**适用场景**：`COUNT()`、`SUM()`、`MAX()` 等聚合函数，或者只查单独的一列且只需要第一个值。

**技术手段**：配合 `select(func.count(...))` 使用。

**返回形态**：应该直接提取为 Python 的基础类型（`int`、`float`、`str`），而不是包含在 `Row` 或实例中。

### ✅ 正确示例

```python
from sqlalchemy import func

# 统计总数
stmt = select(func.count(User.id)).where(User.is_active == True)
with SessionLocal() as session:
    # 正确：使用 .scalar() 直接提取基础类型 int
    total_count: int = session.execute(stmt).scalar()

# 获取最大值
stmt = select(func.max(Order.created_at))
max_time = session.execute(stmt).scalar()

# 异步场景同理
async with async_session() as session:
    active_count: int = (await session.execute(
        select(func.count()).select_from(User).where(User.is_active == True)
    )).scalar()
```

### ❌ 严禁操作

```python
# 致命错误：用 .scalars().all() 取聚合值
count_row = session.execute(stmt).scalars().all()
# 灾难后果：拿到的是类似 [10] 的列表，还要取 [0]，代码极其丑陋且容易出错

# 致命错误：用 .all() 取聚合值
row = session.execute(stmt).all()
# 灾难后果：拿到的是类似 [(10,)] 的 Row 包裹，类型污染，隐式 bug 源头
```

---

## 三种策略的决策速查表

| 需求 | 策略 | 代码模式 | 返回类型 |
|------|------|---------|---------|
| 只需要几个字段，不需要 ORM 能力 | 策略一：列投影 | `select(Col.a, Col.b)` + `.all()` | `list[Row]` |
| 需要 ORM 对象，但表中存在大字段 | 策略二：部分加载 | `select(Model).options(load_only(...), raiseload('*'))` | ORM 实例（残缺） |
| 聚合计算或单列单值 | 策略三：标量提取 | `select(func.count(...))` + `.scalar()` | `int` / `float` / `str` |
| 需要完整的 ORM 实体对象 | 完整加载（少数情况） | `select(Model)` + `.scalars().all()` | `list[Model]` |

---

## 底层性能收益复盘

当严格执行上述规范时，将获得以下三层性能飞跃：

1. **网络 I/O 降维**：避免了将数据库服务器上的大字段（如 50KB 的 JSON）通过网络传输到应用服务器。在千兆网络中，10000 行 50KB/行的数据传输从 ~4 秒降到 ~0.1 秒（仅传索引列）。

2. **数据库 Buffer Pool 命中率提升**：如果查询只涉及索引列（如 `SELECT id, name`，且 `(id, name)` 有联合索引），数据库可以直接走 Index-Only Scan，完全避免访问真实的表数据文件（Heap），查询耗时从毫秒级降到微秒级。

3. **Python 进程内存防线**：`Row` 对象在 C 层实现，内存占用极小，且不加入 Session 的 identity_map；而 ORM 实例包含状态追踪字典和大量属性。查询 10 万条数据时，用 `Row` 可能只需 10MB 内存，用 ORM 实例可能直接吃掉 2GB 内存导致 OOM Kill。这是一条不可逾越的红线——**在生产环境中，OOM 不是性能问题，是服务中断事故。**
