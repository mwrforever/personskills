---
name: best-sqlalchemy-practise
description: SQLAlchemy 2.0+ 生产级最佳实践。当用户编写或 Review 数据库相关代码（模型定义、查询、事务、会话管理、连接池、批量操作、Alembic 迁移）时触发。覆盖 SQLAlchemy ORM/Core、引擎配置、N+1 防护、并发锁、性能调优等所有数据库操作场景。
---

# 启动指令

> **在编写或审查任何一行数据库代码之前，你必须在脑海中逐条过一遍这份 Checklist。跳过任何一条都可能导致生产事故——连接池耗尽、主键冲突、N+1 查询把数据库打挂。这不是建议，这是你在生产环境存活下来的最低要求。**

---

# 核心原则

SQLAlchemy 是一个极其强大的工具，但它不会保护你免受自己错误的影响。它给了你足够的绳子来吊死你自己和你的数据库。以下规则来自超过十年在生产环境中处理千万级日活项目的血泪教训：每一次连接池耗尽、每一次死锁、每一次因 N+1 查询导致的 CPU 飙升至 100%，追根溯源都是一个违反这些规则的编码时刻。

你必须理解每一条规则背后的**为什么**。不理解原理的规则在生产压力下会立刻被遗忘。

---

# 参考文档索引

根据你当前面对的具体问题，读取对应的参考文档获取详细规范、代码示例和底层原理解释：

| 关注点 | 参考文件 | 何时读取 |
|--------|---------|---------|
| 模型定义（字段类型、默认值、索引） | [references/model-layer.md](references/model-layer.md) | 定义/审查 ORM 模型类时 |
| 引擎与连接池配置 | [references/engine-pool.md](references/engine-pool.md) | 配置 `create_engine` / `create_async_engine` 时 |
| Session 生命周期管理 | [references/session-lifecycle.md](references/session-lifecycle.md) | 编写/审查 Session 使用代码时 |
| 查询编写与 Result 处理 | [references/query-result.md](references/query-result.md) | 编写/审查 SELECT 查询时 |
| 按需列加载与内存管控 | [references/column-loading.md](references/column-loading.md) | 涉及大字段、投影查询、`load_only`、聚合查询时 |
| 关系加载与 N+1 防护 | [references/relationship-loading.md](references/relationship-loading.md) | 涉及 relationship 访问、性能优化时 |
| 事务、并发与锁 | [references/transaction-locking.md](references/transaction-locking.md) | 涉及事务边界、悲观锁/乐观锁时 |
| 批量操作性能调优 | [references/batch-operations.md](references/batch-operations.md) | 涉及批量 INSERT/UPDATE/DELETE 时 |
| Alembic 迁移规范 | [references/alembic-migrations.md](references/alembic-migrations.md) | 编写/审查数据库迁移脚本时 |

**重要**：当你同时涉及多个关注点时（例如审查一个包含模型定义、查询和 Session 管理的完整模块），你必须读取所有相关的参考文件。不要只读一篇就跳过其余的。

---

# 最终 Checklist — 编写/审查数据库代码时必须逐条确认

在提交任何包含数据库操作的代码之前，确保以下每一条的答案都是"是"：

- [ ] 所有模型使用 `DeclarativeBase` + `Mapped[]` 体系？（没有 `Column()` 没有 `declarative_base()`）
- [ ] 时间戳字段使用 `server_default=func.now()`？（没有 Python 端 `default=datetime.now`）
- [ ] 每个表的索引不超过 5 个？（没有冗余或重复索引）
- [ ] 连接池配置经过数学验证？（`(pool_size + max_overflow) × workers ≤ db_max × 0.8`）
- [ ] `pool_pre_ping=True` 已设置？（没有例外）
- [ ] 每个请求使用独立的 Session？（没有全局 Session、没有模块级 Session）
- [ ] Session 使用 `async with` 上下文管理器？（没有手动 `commit()` / `rollback()` / `close()`）
- [ ] 所有查询使用 `select()` + `session.execute()`？（没有 `session.query()`）
- [ ] 唯一约束查询使用 `scalar_one_or_none()`？（没有 `.first()` 静默吞下数据问题）
- [ ] 列表/报表查询使用列投影 `select(User.id, User.name)` 而非 `select(User)`？（避免全量 ORM 对象实例化）
- [ ] `load_only()` 使用场景是否同时搭配了 `raiseload('*')`？（防止静默触发补全查询）
- [ ] 聚合查询（COUNT/SUM/MAX）使用 `.scalar()` 直接提取标量？（没有 `.all()` 或 `.scalars().all()` 返回包裹类型）
- [ ] 所有关系默认 `lazy="raise"`？（在 dev/staging 环境强制暴露 N+1）
- [ ] Eager loading 使用 `selectinload`（90% 场景）或 `joinedload`（仅一对一）？（没有隐式 lazy load）
- [ ] 没有在循环中访问关系属性？（所有需要的关联数据已通过 options 预加载）
- [ ] 悲观锁使用 `skip_locked`（队列）或 `nowait`（用户交互）？（没有默认的阻塞等待）
- [ ] 持有锁时不执行任何外部 IO？（网络调用、文件操作、邮件发送都在事务外）
- [ ] 批量写入 > 10000 行时使用 Core 或 COPY 协议？（没有用 ORM add_all 硬扛）
- [ ] 批量 UPDATE/DELETE 使用 Core 语句？（没有逐行加载 ORM 对象再修改）
- [ ] Alembic 迁移脚本已经人工 Review？（没有盲信 `--autogenerate`）
- [ ] 大表 DDL 变更使用了 CONCURRENTLY 或分步策略？（没有直接 ALTER TABLE）
- [ ] 迁移文件包含可执行的 `downgrade()`？（没有 `pass` 占位）

**如果上面任何一条的答案是"否"，代码不允许合并到主分支。这不是代码风格问题，这是生产安全基线。**

---

# 紧急故障响应速查

| 症状 | 最可能的原因 | 参考文件 |
|------|-------------|---------|
| 数据库 CPU 100% | N+1 查询或缺失索引 | [relationship-loading](references/relationship-loading.md)、[model-layer](references/model-layer.md) |
| `QueuePool limit reached` | 连接池配置不足或 Session 泄漏 | [engine-pool](references/engine-pool.md)、[session-lifecycle](references/session-lifecycle.md) |
| `NoResultFound` 或 `MultipleResultsFound` | Result 方法误用 | [query-result](references/query-result.md) |
| 死锁（deadlock detected） | 锁获取顺序不一致或持锁超时 | [transaction-locking](references/transaction-locking.md) |
| 迁移后数据错误 | autogenerate 盲区未被检查 | [alembic-migrations](references/alembic-migrations.md) |
| 批量写入超时 | ORM 处理大量数据 | [batch-operations](references/batch-operations.md) |
| `InvalidRequestError` on lazy load | 关系未预加载 | [relationship-loading](references/relationship-loading.md) |
| `InvalidRequestError` on deferred column | `load_only` 未搭配 `raiseload` | [column-loading](references/column-loading.md) |
| 应用 OOM / 内存飙升 | `SELECT *` 加载大字段或大量 ORM 实例 | [column-loading](references/column-loading.md) |
