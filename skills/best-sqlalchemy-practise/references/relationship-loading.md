# 五、关系加载与 N+1 灾难防护

> **这是整份规范中最重要的章节。N+1 查询是导致数据库 CPU 飙升至 100% 的头号元凶。在所有数据库性能事故中，超过 60% 根因是 N+1 加载。**

## 5.1 四大加载策略的生产级选择标准

### `selectinload` — 默认首选，90% 场景的最佳方案

```python
# 发出的 SQL：
# SELECT * FROM user WHERE ...
# SELECT * FROM order WHERE order.user_id IN (user_id_1, user_id_2, ...)

users = (await session.execute(
    select(User).options(selectinload(User.orders))
)).scalars().unique().all()
```

**适用场景**：一对多、多对多关系。父对象数量在 1-10000 之间。子表行数在百万级以内。

**底层原理**：`selectinload` 用一条独立的 `SELECT ... IN (...)` 语句加载所有关联对象。SQLAlchemy 在内存中组装关联关系。它只用 2 条 SQL 完成加载（1 条父表 + 1 条子表），不产生笛卡尔积。

**警告**：当父对象非常多（>50000）时，IN 列表变得巨大，数据库可能拒绝执行。此时需要使用分批策略。

### `joinedload` — 谨慎使用，笛卡尔积是隐藏杀手

```python
# 发出的 SQL：
# SELECT user.*, order.* FROM user LEFT JOIN order ON user.id = order.user_id

users = (await session.execute(
    select(User).options(joinedload(User.orders))
)).scalars().unique().all()
```

**适用场景**：一对一关系，或者一对多但子表行数极少（每个父对象 ≤3 个子对象）。

**警告 — 笛卡尔积爆炸**：如果 User 有 5 个 Order 和 3 个 Address，同时用 `joinedload` 加载这两个关系：
```python
# 致命错误：同时 joinedload 两个一对多关系
select(User).options(
    joinedload(User.orders),    # ← 不要同时 joinedload
    joinedload(User.addresses), # ← 多个一对多关系
)
```
生成的 SQL 将 User × Order × Address 做 JOIN，结果行数 = `N_users × N_orders × N_addresses`。100 个用户 × 10 个订单 × 5 个地址 = 5000 行结果集，但实际只有 100 + 1000 + 500 条有效数据。多余的 3500 行全部是网络传输 + Python 解序列化的纯浪费。

### `subqueryload` — 不要再用了

```python
# 只在一种情况下考虑：MySQL 的 SELECT ... FOR UPDATE 不能与 JOIN 一起用
select(User).options(subqueryload(User.orders))
```

**底层原理**：`subqueryload` 用子查询加载关联对象。它的 SQL 形态是 `SELECT ... FROM related_table WHERE related_table.fk IN (SELECT pk FROM main_table WHERE ...)`。与 `selectinload` 相比，它多了一层嵌套子查询，MySQL 优化器对此处理极差，PostgreSQL 也会比 IN 列表慢 10-30%。除非你正在使用 MySQL 的 `FOR UPDATE`（它不支持 JOIN 中的锁定），否则永远用 `selectinload`。

### `raiseload` — 开发环境的守护神

### ✅ 强制规范

```python
# 全局配置 — 所有关系默认 raise on lazy load
class Base(DeclarativeBase):
    pass

# 为所有 relationship 设置 lazy="raise"
class User(Base):
    __tablename__ = "user"
    orders = relationship("Order", lazy="raise", back_populates="user")

# 或者用事件全局设置（推荐）
from sqlalchemy import event

@event.listens_for(Base, "mapper_configured", propagate=True)
def _setup_raiseload(mapper, class_):
    for rel in mapper.relationships:
        if rel.lazy_loadopt is None:
            rel.lazy = "raise"
```

### ❌ 严禁操作

```python
class User(Base):
    orders = relationship("Order", lazy="select")  # ← 默认值，允许隐式 lazy load

# 然后在模板中
{% for user in users %}
    {{ user.name }}
    {% for order in user.orders %}  <!-- 每次循环触发一条 SELECT！-->
        {{ order.id }}
    {% endfor %}
{% endfor %}
# 如果有 100 个用户，这会产生 1 + 100 条 SQL
# 更糟糕的是开发者在本地测试用 10 个用户，完全察觉不到问题
# 等到生产环境 10000 个用户时，数据库直接被打挂
```

**底层原理解释**：`lazy="raise"` 让任何意外的延迟加载立即抛出 `InvalidRequestError`，在开发阶段就暴露问题。等到了生产环境再把 `lazy="raise"` 降级为强制指定 eager loading 策略，而不是依赖隐式行为。这是"fail fast"理念在数据库层的实践——让错误在离开发者最近的地方爆炸，而不是在凌晨 3 点的生产环境沉默地杀死数据库。

---

## 5.2 禁止在循环中访问关系属性

### ✅ 强制规范

```python
# 提前用 selectinload 加载好所有需要的关系
users = (await session.execute(
    select(User)
    .where(User.is_active == True)
    .options(selectinload(User.orders).selectinload(Order.items))
)).scalars().unique().all()

for user in users:
    for order in user.orders:       # 访问已加载的关系 — 无额外 SQL
        for item in order.items:    # 访问已加载的关系 — 无额外 SQL
            process(user, order, item)
```

### ❌ 严禁操作

```python
# 致命错误：循环中逐个访问关系
users = (await session.execute(select(User))).scalars().all()

for user in users:
    orders = user.orders  # 第一次访问触发 lazy load → SELECT ... FROM order WHERE user_id = ?
    for order in orders:
        items = order.items  # 又触发 lazy load → SELECT ... FROM item WHERE order_id = ?
        # 如果 users=100, orders/user=5 → 1 + 100 + 500 = 601 条 SQL
        # 生产故障：数据库 CPU 从 15% 飙升至 98%，所有 API 超时
```

---

## 5.3 多层级关系预加载

### ✅ 强制规范

```python
# 使用链式 selectinload 预加载多层关系
users = (await session.execute(
    select(User).options(
        selectinload(User.orders)
        .selectinload(Order.items)       # 第 2 层
        .selectinload(Item.vendor),      # 第 3 层
        selectinload(User.profile),       # 第 2 个顶层关系
    )
)).scalars().unique().all()
```
