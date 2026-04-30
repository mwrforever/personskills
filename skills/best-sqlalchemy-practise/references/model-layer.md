# 一、模型定义层

## 1.1 强制使用 2.0+ 声明体系

### ✅ 强制规范

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship
from sqlalchemy import Integer, String, DateTime, func, Index
import datetime

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    email: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),  # 数据库服务器时间
        nullable=False,
    )
```

### ❌ 严禁操作

```python
# 绝对禁止使用旧版 declarative_base()
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base()

# 绝对禁止使用 Column() 而不使用 Mapped[] 类型注解
class User(Base):
    __tablename__ = "user"
    id = Column(Integer, primary_key=True)  # 1.x 遗产，类型检查器无法推导类型
    name = Column(String)  # 连长度都不指定，等着数据库隐式转换
```

**底层原理解释**：SQLAlchemy 1.x 的 `Column()` 体系不携带类型信息给静态分析器（mypy/pyright），导致 IDE 自动补全失效、类型检查形同虚设。`Mapped[]` 注解让类型检查器能够推导出 `User.name` 是 `str` 类型，`User.created_at` 是 `datetime.datetime` 类型，从根源上消除了一整类因类型错误导致的运行时崩溃。

---

## 1.2 字段默认值：`server_default` vs `default` 的生死抉择

### ✅ 强制规范

```python
# 始终用 server_default 让数据库生成时间戳
created_at: Mapped[datetime.datetime] = mapped_column(
    DateTime(timezone=True),
    server_default=func.now(),
)

updated_at: Mapped[datetime.datetime] = mapped_column(
    DateTime(timezone=True),
    server_default=func.now(),
    onupdate=func.now(),  # server_onupdate 在部分方言需要手动实现
)

# 布尔字段的数据库级默认值
is_active: Mapped[bool] = mapped_column(
    Boolean,
    server_default="true",  # 字符串，由数据库解析
    nullable=False,
)
```

### ❌ 严禁操作

```python
# 致命错误：用 Python 端 default 生成时间戳
created_at = mapped_column(DateTime, default=datetime.datetime.now)
# 生产故障：(1) 不同应用服务器时钟偏差导致数据不一致
#          (2) ORM 生成的时间与数据库实际写入时间有微妙延迟
#          (3) 批量 INSERT 时 Python 端时间完全相同，排序失去意义

# 同样致命：default=func.now()
created_at = mapped_column(DateTime, default=func.now)
# 这会在 INSERT 的 VALUES 子句中嵌入 now() 调用，但 SQLAlchemy 可能
# 无法正确将它标记为服务端默认，导致后续 session.refresh() 行为异常
```

**底层原理解释**：多服务器部署时时钟偏差（clock skew）不可避免。NTP 通常只能把偏差控制在 ±5ms 到 ±100ms 之间。如果应用 A（时钟快 50ms）插入一行，应用 B（时钟慢 30ms）插入另一行，`ORDER BY created_at` 会返回与物理插入顺序相反的结果。`server_default=func.now()` 使用数据库服务器的单一时钟源（WAL 写入时刻），保证单调递增和全局一致性。

---

## 1.3 索引策略：不是每个 WHERE 字段都需要一个索引

### ✅ 强制规范

```python
from sqlalchemy import Index, column

class Order(Base):
    __tablename__ = "order"
    __table_args__ = (
        # 联合索引：最左前缀原则 — 查询 WHERE user_id 会使用此索引，
        # 但查询 WHERE status 不会
        Index("ix_order_user_status", "user_id", "status"),

        # 表达式索引：用于函数查询
        Index("ix_order_lower_email", func.lower(User.email)),

        # 部分索引（PostgreSQL）：仅索引活跃订单
        Index(
            "ix_active_order_user",
            "user_id",
            postgresql_where=(column("status") == "active"),
        ),
    )

    id: Mapped[int] = mapped_column(primary_key=True)
    user_id: Mapped[int] = mapped_column(Integer, index=True)  # 单列索引简化写法
    status: Mapped[str] = mapped_column(String(20))
    email: Mapped[str] = mapped_column(String(255))
```

### ❌ 严禁操作

```python
# 为每个列都加 index=True — 这是索引的滥用，不是优化
class AntiPattern(Base):
    __tablename__ = "anti_pattern"
    name: Mapped[str] = mapped_column(String, index=True)
    email: Mapped[str] = mapped_column(String, index=True)
    phone: Mapped[str] = mapped_column(String, index=True)
    address: Mapped[str] = mapped_column(String, index=True)
    city: Mapped[str] = mapped_column(String, index=True)
    # 每个索引都在 INSERT/UPDATE/DELETE 时产生额外写入开销
    # 6 个索引 = INSERT 性能下降 3-5 倍，存储空间膨胀 50-100%

# 创建冗余索引
Index("ix_a", "a"),           # ← 联合索引已经覆盖了它
Index("ix_a_b", "a", "b"),    # ← 冗余：任何用到 ix_a 的查询都能用 ix_a_b
```

**底层原理解释**：联合索引的最左前缀原则意味着 `INDEX(a, b, c)` 可以服务于 `WHERE a = ?` 和 `WHERE a = ? AND b = ?`，但不能服务于 `WHERE b = ?` 或 `WHERE c = ?`。索引是 B+ 树，每个新索引就是一棵新树。每次 INSERT 都要同时更新所有索引树。在生产环境中，索引设计必须在查询速度和写入性能之间做权衡。一个好的经验法则：为每个表建立不超过 3-5 个精心设计的索引。
