# 三、会话生命周期管理

## 3.1 全局 Session：生产环境中的定时炸弹

### ✅ 强制规范

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

# 工厂模式 — 按请求创建 Session
AsyncSessionFactory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,  # 提交后不使对象过期，避免延迟加载
    autoflush=False,         # 手动控制 flush 时机
)

# 每个业务操作拥有独立的 Session
async def get_user(user_id: int) -> User:
    async with AsyncSessionFactory() as session:
        return await session.get(User, user_id)
```

### ❌ 严禁操作

```python
# 绝对禁止：模块级全局 Session
session = Session()  # ← 这是定时炸弹

def get_user(user_id):
    return session.query(User).get(user_id)
    # 生产故障：(1) 线程不安全 — 多线程同时操作同一个 session 导致状态污染
    #          (2) 事务边界模糊 — 谁负责 commit？谁负责 rollback？
    #          (3) 内存泄漏 — identity_map 无限膨胀，从不清理
    #          (4) 连接泄漏 — session 持有一个连接但从不归还连接池
```

**底层原理解释**：SQLAlchemy 的 `Session` 遵循 Unit of Work 模式。一个 Session 实例代表一个逻辑业务操作单元——一个请求、一个后台任务周期。Session 内部维护着 identity_map（主键到对象的映射缓存）和脏对象追踪列表。全局共享 Session 意味着所有请求共享同一个 identity_map：一个请求查出来的 10000 条数据会留在内存中，另一个请求的 `session.flush()` 会把前一个请求意外修改的对象也一并写入数据库。在生产环境中，这要么导致内存溢出（OOM），要么导致数据错误。

---

## 3.2 上下文管理器：自动回滚的异常安全网

### ✅ 强制规范

```python
async def transfer_money(from_id: int, to_id: int, amount: float):
    async with AsyncSessionFactory() as session:
        async with session.begin():  # 自动 commit/rollback
            from_acct = await session.get(Account, from_id, with_for_update=True)
            to_acct = await session.get(Account, to_id, with_for_update=True)

            if from_acct.balance < amount:
                raise ValueError("余额不足")

            from_acct.balance -= amount
            to_acct.balance += amount
        # session.begin() 上下文退出时：
        # 无异常 → 自动 commit
        # 有异常 → 自动 rollback
```

### ❌ 严禁操作

```python
# 致命错误：手动管理事务
session = Session()
try:
    user = session.get(User, 1)
    user.name = "new name"
    session.commit()  # 如果这里抛异常，session 未被正确清理
except Exception:
    session.rollback()  # rollback 后 session 仍处于脏状态
    raise
# 忘记 session.close() → 连接泄漏！
```

**底层原理解释**：`session.begin()` 返回的上下文管理器在 `__exit__` 方法中检查是否有异常发生。无异常时自动调用 `session.commit()`；有异常时自动调用 `session.rollback()`。而 `async with AsyncSessionFactory() as session` 在退出时自动调用 `session.close()`，将连接归还连接池。这构成了两层嵌套的安全网：外层确保连接归还，内层确保事务正确终止。手动管理时，开发者总是漏掉某些边缘情况（如 KeyboardInterrupt、asyncio.CancelledError 等）。

---

## 3.3 Web 框架集成范式

### FastAPI 依赖注入（正确范式）

```python
from fastapi import Depends
from typing import AsyncGenerator

async def get_session() -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSessionFactory() as session:
        yield session
        # 响应返回后自动关闭 session

@app.get("/users/{user_id}")
async def read_user(
    user_id: int,
    session: AsyncSession = Depends(get_session),
):
    user = await session.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404)
    return user
```

### Flask 中间件（正确范式）

```python
from flask import g

def init_db_middleware(app):
    @app.before_request
    def create_session():
        g.session = SessionFactory()

    @app.teardown_request
    def close_session(exception=None):
        session = g.pop("session", None)
        if session is not None:
            if exception is None:
                session.commit()
            else:
                session.rollback()
            session.close()
```

### ❌ 严禁操作

```python
# 致命错误：在 FastAPI 的模块级别创建 Session
session = Session()
@app.get("/")
def root():
    return session.execute(...)
# 所有请求共享一个 Session — 参见 3.1 节的所有故障
```
