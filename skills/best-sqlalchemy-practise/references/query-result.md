# 四、查询与 Result 处理

## 4.1 `session.query()` 已死

### ✅ 强制规范

```python
# 唯一正确的查询方式
stmt = select(User).where(User.email == "alice@example.com")
result = await session.execute(stmt)
user = result.scalar_one_or_none()
```

### ❌ 严禁操作

```python
# 绝对禁止：1.x 风格的 query()
user = session.query(User).filter(User.email == "alice@example.com").first()
# 原因：(1) 类型检查器无法推导返回类型
#       (2) 与 async session 不兼容
#       (3) 被 SQLAlchemy 官方标记为 deprecated，将在 3.0 中移除
#       (4) 隐式生成 FROM 子句，多表 join 时难以理解生成的 SQL
```

---

## 4.2 Result 方法的精确使用场景

| 方法 | 返回值 | 使用场景 |
|------|--------|----------|
| `result.scalars().all()` | `list[Model]` | 获取所有行的 ORM 对象列表 |
| `result.scalars().first()` | `Model \| None` | 获取第一行，不要求唯一性 |
| `result.scalar_one()` | `Model` | 期望恰好 1 行；0 行抛 `NoResultFound`，>1 行抛 `MultipleResultsFound` |
| `result.scalar_one_or_none()` | `Model \| None` | 期望 0 或 1 行；>1 行抛 `MultipleResultsFound` |
| `result.scalars().unique()` | 去重后的列表 | joinedload 查询后去重 |

### ✅ 强制规范

```python
# 按主键查询 — 必须用 session.get()
user = await session.get(User, user_id)  # 使用 identity_map 缓存，O(1)

# 唯一约束查询 — 用 scalar_one_or_none
stmt = select(User).where(User.email == email)
user = await session.execute(stmt).scalar_one_or_none()  # 语义明确：最多一行

# 有且仅有一行 — 用 scalar_one
stmt = select(func.count()).select_from(User)
count = await session.execute(stmt).scalar_one()  # 聚集查询永远只有一行

# 多行列表 — 用 scalars().all()
stmt = select(User).where(User.is_active == True)
users = (await session.execute(stmt)).scalars().all()
```

### ❌ 严禁操作

```python
# 致命错误：用 .first() 检查唯一性
user = (await session.execute(select(User).where(User.email == email))).scalars().first()
# 如果 email 有唯一约束但数据中有脏数据，.first() 静默返回第一条
# scalar_one_or_none() 会抛出 MultipleResultsFound，暴露数据问题

# 致命错误：用 scalar_one() 查询可能不存在的结果
user = (await session.execute(select(User).where(User.id == 999))).scalar_one()
# 用户不存在 → NoResultFound 异常。应该用 scalar_one_or_none() + 手动 None 检查

# 致命错误：在结果集上直接使用 .all() 而不先 .scalars()
rows = (await session.execute(select(User))).all()
# rows 是 list[Row[Tuple[User]]]，不是 list[User]
# 每个元素是 Row 对象 — 类型注解不管用，运行时才发现问题
```

**底层原理解释**：`session.execute()` 返回 `CursorResult`，其中每一行是 `Row` 对象（类似命名元组）。`Row` 可以包含多个实体（如在 join 查询中）。`.scalars()` 方法从每个 Row 中提取第一个标量值，返回 `ScalarResult`。跳到正确的方法不仅是类型问题——`results.all()` 和 `results.scalars().all()` 返回的是完全不同的数据结构，混淆它们会导致模板渲染崩溃、JSON 序列化失败等生产故障。
