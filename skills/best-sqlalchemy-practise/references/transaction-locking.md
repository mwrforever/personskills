# 六、事务、并发与锁机制

## 6.1 显式事务边界的 SAVEPOINT 应用

### ✅ 强制规范

```python
async def create_order_with_items(user_id: int, items: list[dict]):
    async with AsyncSessionFactory() as session:
        async with session.begin():  # 最外层事务
            order = Order(user_id=user_id)
            session.add(order)
            await session.flush()  # 获取 order.id

            for item_data in items:
                async with session.begin_nested():  # SAVEPOINT
                    # 如果某个 item 创建失败，只回滚这个 item 的保存点
                    # 最外层事务不受影响
                    try:
                        item = OrderItem(
                            order_id=order.id,
                            product_id=item_data["product_id"],
                            quantity=item_data["quantity"],
                        )
                        session.add(item)
                        await session.flush()
                    except IntegrityError:
                        # 回滚到保存点，记录日志，继续处理下一个 item
                        pass
            # 会话结束时统一 commit
```

**底层原理解释**：`session.begin_nested()` 发出一个 PostgreSQL `SAVEPOINT sp_...` 命令。SAVEPOINT 允许在事务内部创建一个可以独立回滚的检查点。在高并发环境下，这是处理部分失败的最佳实践：你不需要回滚整个请求逻辑，只需回滚失败的子操作。但请注意——SAVEPOINT 不能跨 connection 使用，它绑定在一个数据库连接上的活跃事务中。

---

## 6.2 悲观锁：`with_for_update()` 和 `skip_locked`

### ✅ 强制规范 — 消息队列消费

```python
from sqlalchemy import select, update

async def consume_tasks(worker_id: int, batch_size: int = 10):
    async with AsyncSessionFactory() as session:
        async with session.begin():
            # skip_locked=True：跳过已被其他 worker 锁定的行
            # 多个 worker 并发消费同一队列，不会互相阻塞
            stmt = (
                select(Task)
                .where(Task.status == "pending")
                .order_by(Task.created_at)
                .limit(batch_size)
                .with_for_update(skip_locked=True)
            )
            tasks = (await session.execute(stmt)).scalars().all()

            for task in tasks:
                task.status = "processing"
                task.worker_id = worker_id
            # session.begin() 退出时自动 commit，释放锁
```

### ✅ 强制规范 — 库存扣减

```python
async def deduct_stock(product_id: int, quantity: int) -> bool:
    async with AsyncSessionFactory() as session:
        async with session.begin():
            product = await session.get(
                Product, product_id,
                with_for_update=True,
            )

            if product.stock < quantity:
                return False

            product.stock -= quantity
            return True
```

### ❌ 严禁操作

```python
# 致命错误：SELECT ... FOR UPDATE 但没有使用索引
stmt = select(Product).where(Product.name == name).with_for_update()
# 如果 name 列没有索引，PostgreSQL 锁住所有被顺序扫描扫描到的行
# 包括那些不匹配 name == name 的行！这是"幻读锁"现象
# 导致完全不相关的查询也被阻塞

# 致命错误：长事务中持有 FOR UPDATE 锁
async with session.begin():
    task = await session.get(Task, id, with_for_update=True)
    await external_api_call()  # ← 持有锁时调用外部服务
    await send_email()         # ← 网络 IO 期间整个表被锁
# 事务中永远先 IO，再锁；不要在持有锁时做任何 IO
```

**底层原理解释**：PostgreSQL 的 `SELECT ... FOR UPDATE` 在行级加排他锁。`skip_locked` 是 PostgreSQL 9.5+ 的特性，它跳过任何已经被其他事务锁定的行——完美适合消息队列的"领取并处理"模式。`nowait` 是另一个选项：碰到被锁定的行立刻抛出异常，适用于用户交互场景（如库存检查），因为用户宁愿看到"请稍后重试"也不希望等待 30 秒。

---

## 6.3 乐观锁：版本号控制

### ✅ 强制规范

```python
from sqlalchemy import Integer

class Document(Base):
    __tablename__ = "document"
    id: Mapped[int] = mapped_column(primary_key=True)
    content: Mapped[str] = mapped_column(Text)
    version: Mapped[int] = mapped_column(Integer, default=1, nullable=False)

async def update_document(session: AsyncSession, doc_id: int, new_content: str,
                          current_version: int) -> bool:
    # 使用版本号实现乐观锁
    stmt = (
        update(Document)
        .where(Document.id == doc_id)
        .where(Document.version == current_version)  # ← 乐观锁检查
        .values(content=new_content, version=Document.version + 1)
    )
    result = await session.execute(stmt)
    return result.rowcount > 0  # rowcount=0 表示版本号已过期，发生了并发修改
```

**底层原理解释**：悲观锁（`FOR UPDATE`）假设并发冲突会发生，提前加锁。乐观锁（版本号）假设并发冲突很少发生，提交时再检查。在低冲突场景（如文档编辑），乐观锁避免锁开销和死锁风险；在高冲突场景（如秒杀库存），悲观锁通过排队机制避免大量重试。选择标准：预期冲突率 > 10% 时用悲观锁，< 1% 时用乐观锁。
