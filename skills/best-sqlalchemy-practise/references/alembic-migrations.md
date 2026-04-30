# 八、Alembic 迁移生产规范

## 8.1 手动改表是生产事故

```
# 绝对禁止以下操作：
$ psql -h production-db -c "ALTER TABLE user ADD COLUMN ..."
# 这会导致：
# (1) 没有版本记录 — 其他环境不知道这个变更
# (2) 没有回滚路径 — 出问题只能手工恢复
# (3) 绕过代码审查 — 你是唯一知道这个变更存在的人
# (4) 锁表 —— 生产流量中断
```

### ✅ 强制规范

```bash
# 唯一的正确流程
alembic revision --autogenerate -m "add_user_last_login_column"
# 然后 —— 必须 —— 阅读生成的迁移文件
# 确认无误后
alembic upgrade head
```

---

## 8.2 `autogenerate` 的盲区：你必须人工检查

### autogenerate 能检测到

- 新增/删除的表
- 新增/删除的列
- 列类型、nullable、server_default 的变更
- 新增/删除的外键

### autogenerate **无法**检测到（你可能在线上部署了一个空洞的迁移）

| 盲区 | 后果 |
|------|------|
| 索引改名 | 旧索引不删除，新索引被创建，性能混淆 |
| 约束改名 | 旧约束残留，可能阻塞后续 DDL |
| 枚举值新增 | PostgreSQL ALTER TYPE ADD VALUE 不会被 autogenerate |
| 表改名 | autogenerate 会生成 drop + create，数据丢失 |
| 部分索引变更 | WHERE 子句变化 autogenerate 看不到 |
| 自定义 DDL 触发器/函数 | 完全不可见 |

### ✅ 强制规范

```python
# 每次 alembic revision --autogenerate 之后，你必须：
# 1. 逐行阅读生成的 upgrade() 和 downgrade()
# 2. 检查是否有意外的 DROP TABLE / DROP COLUMN
# 3. 验证外键命名是否符合项目规范
# 4. 确认索引策略正确（是否有冗余索引？）
# 5. 在 staging 环境执行一次 dry-run
```

---

## 8.3 在线 DDL 变更的安全策略

### 大数据量表变更的危险操作

```python
# 危险操作：给百万级表加 NOT NULL 列
# ALTER TABLE "order" ADD COLUMN "tracking_id" VARCHAR NOT NULL;
# 这会导致 PostgreSQL 对整个表加 AccessExclusiveLock，阻塞所有读写

# ✅ 安全策略：分步执行
def upgrade():
    # 步骤 1：添加 nullable 列（瞬间完成，无锁）
    op.add_column("order", sa.Column("tracking_id", sa.String(), nullable=True))

def upgrade_2():
    # 步骤 2：分批填充数据（在业务低峰期执行）
    op.execute("""
        UPDATE "order" SET tracking_id = gen_random_uuid()
        WHERE tracking_id IS NULL AND id BETWEEN :start AND :end
    """)

def upgrade_3():
    # 步骤 3：在上一步确认所有数据填充完毕后，添加 NOT NULL 约束
    op.alter_column("order", "tracking_id", nullable=False)
```

### ✅ 强制规范

```python
# 对于大表（> 100万行），以下操作使用 ALGORITHM=INPLACE（MySQL）或 CONCURRENTLY（PostgreSQL）

# PostgreSQL 创建索引的正确方式
op.execute("CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_order_user ON \"order\" (user_id)")
# CONCURRENTLY = 不阻塞写入，但需要额外扫描一次表，且不能在事务中执行

# MySQL 的正确方式
op.execute("ALTER TABLE `order` ADD INDEX ix_order_user (user_id), ALGORITHM=INPLACE, LOCK=NONE")
```

### ❌ 严禁操作

```python
# 致命错误：在高流量表上直接创建索引
op.create_index("ix_order_user", "order", ["user_id"])
# 生成的 SQL：CREATE INDEX ix_order_user ON "order" (user_id);
# 不用 CONCURRENTLY → 整个表被 ShareLock 锁住 → 所有 INSERT 被阻塞

# 致命错误：在单个事务中修改多列
def upgrade():
    op.alter_column("order", "status", ...)  # ←
    op.alter_column("order", "amount", ...)  # ← 同一个事务中多次 ALTER TABLE
    op.alter_column("order", "currency", ...)# ← 每次都需要 AccessExclusiveLock
# 正确做法：每个 ALTER TABLE 独立一个迁移版本
```

**底层原理解释**：PostgreSQL 的 MVCC 模型要求某些 DDL 操作获取 `AccessExclusiveLock`——这是最重级别的表锁，与任何其他访问（包括 SELECT）冲突。在线 DDL 技巧（`CREATE INDEX CONCURRENTLY`、分步加 NOT NULL 列等）的原理是将一次需要重锁的操作拆分为多个轻锁步骤，使得每个步骤只阻塞极短时间。
