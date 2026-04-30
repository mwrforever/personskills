# 本地缓存选型：CHM vs Caffeine 边界划分

## ConcurrentHashMap (CHM)

### 本质
**只是一个线程安全的 Map，不是缓存框架**。它不具备任何缓存管理能力。

### 适用场景
极小规模的纯运行时状态维护：
- Spring 单例 Bean 内部维护一个大小固定的任务注册表（几十个元素级别）
- 线程安全的配置参数共享
- 临时性的、非业务级的数据交换

### 致命缺陷

| 缺陷 | 后果 |
|------|------|
| 无过期机制 | Key 永远存在，容易内存泄漏 |
| 无淘汰策略 | 无限制增长，容易 OOM |
| 无缓存统计 | 无法监控命中率，无法优化 |
| 无加载策略 | 需要手动 `put`，无法自动从数据源加载 |
| 无刷新机制 | 无法自动刷新过期数据 |

### 误用案例

```java
// 错误用法 - CHM 被当作缓存使用
private static final Map<String, User> USER_CACHE = new ConcurrentHashMap<>();

public User getUser(String id) {
    User user = USER_CACHE.get(id);
    if (user == null) {
        user = db.query(id);
        USER_CACHE.put(id, user); // 永远不清理，内存持续增长
    }
    return user;
}
```

---

## Caffeine

### 本质
近乎完美的本地缓存框架，基于 **W-TinyLFU** 算法（WeakLRU 变种），命中率远超 Guava Cache。

### 适用场景
所有需要承载一定并发量、需要控制内存上限的业务级本地缓存：
- 字典表、配置项缓存
- 极其热点的只读数据（如秒杀活动配置）
- 需要控制内存上限的缓存

### 核心配置

```java
LoadingCache<String, User> cache = Caffeine.newBuilder()
    // 容量控制：超过 10000 条时基于 W-TinyLFU 淘汰
    .maximumSize(10_000)
    // 写入后 5 分钟过期
    .expireAfterWrite(5, TimeUnit.MINUTES)
    // 异步刷新：Key 过期时不阻塞，返回旧值，后台线程更新
    .refreshAfterWrite(5, TimeUnit.MINUTES)
    // 软引用/弱引用：堆内存不足时自动回收
    .softValues()
    // 淘汰监听器
    .removalListener((key, value, cause) -> {
        // cause: EXPLICIT/REPLACED/EXPIRED/EVICTED/SIZE
        System.out.println("Key=" + key + " 被移除，原因=" + cause);
    })
    .build(key -> loadFromDb(key)); // 异步加载
```

### 高级特性详解

#### 1. `refreshAfterWrite`（防击穿神器）

**问题**：缓存过期时，如果同时有 1000 个请求过来，都去查 DB，导致缓存击穿。

**CHM 方案**：无解，必须等到第一个请求完成才能恢复。

**Caffeine 方案**：
- 过期后，第一个请求拿到旧值（不阻塞）
- 后台线程异步加载新值
- 其他 999 个请求也拿到旧值
- 新值加载完成后，后续请求拿到新值

#### 2. `weakKeys/weakValues/softValues`

| 配置 | 回收策略 | 适用场景 |
|------|----------|----------|
| weakKeys() | Key 被 GC 时回收 | Key 是动态对象，容易内存泄漏 |
| weakValues() | Value 被 GC 时回收 | Value 是临时对象 |
| softValues() | 内存不足时 LRU 回收 | 缓存数据量大，JVM 堆内存紧张 |

**推荐配置**：`.softValues()` —— 在 OOM 前自动回收，不会影响业务线程。

#### 3. `RemovalListener`

常用于：
- 本地缓存失效时通知下游（如发送 Redis 消息）
- 记录淘汰日志
- 清理关联资源

### 性能对比

| 指标 | CHM | Caffeine |
|------|-----|----------|
| 命中率 | 取决于业务逻辑 | W-TinyLFU 优化，命中率极高 |
| 内存控制 | 无 | maximumSize + softValues |
| 过期策略 | 无 | expireAfterWrite/Access |
| 线程安全 | 是 | 是（无锁读 + CAS 写） |
| 异步刷新 | 无 | refreshAfterWrite |

---

## 决策树

```
需要本地缓存？
├── 数据规模 < 100 条，极少变化，且不会持续增长？
│   ├── 是 → CHM（仅作为运行时状态共享）
│   └── 否 → 下一步
├── 需要内存上限控制？
│   ├── 是 → Caffeine
│   └── 否 → 下一步
├── 需要过期/淘汰机制？
│   ├── 是 → Caffeine
│   └── 否 → CHM（但建议重新评估是否真的需要缓存）
└── 需要统计/监控？
    └── Caffeine（提供 `recordStats()`）
```

---

## 常见错误汇总

| 错误用法 | 问题 | 正确做法 |
|----------|------|----------|
| `new ConcurrentHashMap<>()` 作为缓存 | 内存泄漏、无过期 | 使用 Caffeine |
| 只用 `maximumSize` 不设 `expireAfterWrite` | 可能永远占满内存 | 配合过期时间 |
| 不设置 `softValues()` 在大缓存场景 | 堆内存压力 | 添加 `.softValues()` |
| 同步加载 `build(key -> db.find(key))` | 缓存击穿时阻塞 | 异步加载或用 `refreshAfterWrite` |