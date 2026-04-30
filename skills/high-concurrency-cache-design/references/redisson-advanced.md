# Redisson 高级特性全景

## 核心原则：禁止原生 RedisTemplate 执行 Lua

### 禁令

```java
// 禁止 - 每次传输完整 Lua 脚本，网络/CPU 浪费
redisTemplate.execute(new DefaultRedisScript<>(
    "return redis.call('GET', KEYS[1])",
    Long.class,
    Collections.singletonList("mykey")
));
```

### 原因

在极高并发下（10 万 QPS+），原生方式的问题：
1. **网络开销**：每次传递完整 Lua 脚本体（约 200-500 字节），而不是 SHA1 签名（40 字节）
2. **Redis CPU 解析**：每次需要解析 Lua 脚本，而不是直接执行预编译的脚本
3. ** EVALSHA vs EVAL**：Redis 需要检查脚本是否已加载，未加载则需要重新加载

### 正确做法：RScript 预注册

```java
// 1. 应用启动时注册脚本（只执行一次）
RScript script = redissonClient.getScript();
String sha1 = script.load(Md5Hash.hash("my_script.lua"),
    RScript.Mode.READ_ONLY,
    "return redis.call('GET', KEYS[1])");

// 2. 后续调用只传 SHA1 签名
RedissonScriptingCommand<String> command = new RedissonScriptingCommand<>(...);
String result = command.evalSHA(Arrays.asList("mykey"), sha1, ...);
```

**简化写法（Redisson 封装）**：
```java
RScript script = redissonClient.getScript();
// load 会自动计算 SHA1 并注册到 Redis
String sha = script.load(Long.valueOf(1), // 注册到 Redis 的 key
    "return redis.call('GET', KEYS[1])");

// 调用时只需传 SHA1
String result = script.evalSHA(RScript.Mode.READ_ONLY, sha, 
    RScript.ReturnType.VALUE, 
    Collections.singletonList("mykey"));
```

---

## 分布式锁 RLock

### 基本用法

```java
RLock lock = redissonClient.getLock("my:lock:key");
try {
    // 尝试加锁，最多等 10 秒，锁自动 30 秒后释放（看门狗机制）
    boolean acquired = lock.tryLock(10, 30, TimeUnit.SECONDS);
    if (acquired) {
        // 业务逻辑
    }
} finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

### 看门狗机制（关键）

**默认情况（不指定 leaseTime）**：
- 锁自动续期：每 10 秒自动延长 30 秒
- 如果业务执行超过 30 秒，锁不会过期
- 只有调用 `unlock()` 或连接断开时才释放

**错误用法（指定 leaseTime）**：
```java
// 错误！指定 leaseTime 后，看门狗失效
lock.tryLock(10, 5, TimeUnit.SECONDS); // 5 秒后锁过期，业务还没执行完
```
**后果**：业务还在运行，锁已经被其他线程获取，导致并发安全问题。

**正确做法**：
```java
// 正确：不指定 leaseTime，或指定 -1
lock.tryLock(10, -1, TimeUnit.SECONDS); // 看门狗生效
// 或者
lock.tryLock(); // 默认 leaseTime = -1
```

### 红锁 RedissonRedLock（已废弃）

**警告**：Redis 官方已废弃 RedLock 算法。

**原因**：Martin Kleppmann 的严密数学论证 —— 主从切换时的 GC 延迟导致锁互斥失效。

**替代方案**：
- 单 Redis 实例主从：使用普通 `RLock`
- 跨数据源加锁：使用 Zookeeper 或 etcd
- 业务对锁要求极高：使用 RedLock 算法但配合 fencing token

---

### 联锁 RMultiLock

将多个 RLock 组合为一个逻辑锁，确保要么全部加锁成功，要么全部失败。适用于跨多个 Redis 实例的场景。

**用法**：
```java
RLock lock1 = redissonClient.getLock("lock1");
RLock lock2 = redissonClient.getLock("lock2");
RLock lock3 = redissonClient.getLock("lock3");

RMultiLock multiLock = new RedissonMultiLock(lock1, lock2, lock3);

try {
    // 全部加锁，最多等待 30 秒
    multiLock.lock(30, TimeUnit.SECONDS);
    // 业务逻辑
} finally {
    // 释放全部锁
    multiLock.unlock();
}

// 尝试获取（快速失败）
boolean acquired = multiLock.tryLock(10, TimeUnit.SECONDS);
```

**审查点**：
- 联锁不保证原子性（多个独立锁的组合），无法防止部分加锁成功的情况
- 如果某节点宕机，其他节点上的锁不会被自动释放，需要手动清理或依赖 TTL 自动过期
- 高可用要求高的场景，建议使用 Zookeeper/etcd 的跨节点协调

---

### 栅栏锁 RFencedLock

解决分布式锁的"安全边界"问题。普通 RLock 无法防止持锁者崩溃后锁永不释放（需依赖 TTL），而 RFencedLock 通过**Fencing Token**机制确保即使锁过期，下游业务也能识别过期的锁请求。

**核心概念**：
- 每次加锁成功，Redisson 分配一个严格递增的 Token（类似乐观锁 version）
- 业务处理时需要将 Token 写入下游系统（如 DB）
- 下游系统在写入时校验 Token，拒绝过期请求

**用法**：
```java
RFencedLock lock = redissonClient.getFencedLock("inventory:lock");

try {
    long token = lock.lockAndGetToken();
    // 将 token 传给下游业务（如存储到 DB）
    inventoryService.decreaseStock(goodsId, count, token);
} finally {
    lock.unlock();
}
```

**下游校验示例**：
```java
// 数据库表需要额外字段：fenced_token（最新批准的操作 token）
@Update("UPDATE inventory SET stock = stock - #{count}, fenced_token = #{token} WHERE id = #{id} AND fenced_token < #{token}")
int decreaseStock(Long id, Integer count, Long token);
```

**适用场景**：
- 库存扣减、余额操作等**不可重复**的业务
- 需要确保"即使锁过期，旧的写请求也被拒绝"的场景

**注意**：RFencedLock 不支持看门狗机制，leaseTime 必须指定。

---

## 读写锁 RReadWriteLock

### 适用场景

读多写少且对数据一致性要求极高的场景（如商品详情缓存更新）。

### 用法

```java
RReadWriteLock rwLock = redissonClient.getReadWriteLock("my:rw:lock");

// 读锁 - 防止脏写，不互斥读
RLock readLock = rwLock.readLock();
readLock.lock();
try {
    // 读取操作，可以并发
} finally {
    readLock.unlock();
}

// 写锁 - 互斥读写
RLock writeLock = rwLock.writeLock();
writeLock.lock();
try {
    // 写入操作，排他
} finally {
    writeLock.unlock();
}
```

### 审查点

1. **强制使用读锁防击穿**：多个线程同时读，读锁不互斥
2. **写锁防脏写**：写操作时独占，其他线程无法读/写
3. **写锁降级为读锁**：可以 —— `writeLock.lock(); ... readLock.lock(); writeLock.unlock();`
4. **读锁不能升级为写锁**：会死锁

---

## 限流器 RRateLimiter

### 底层原理

底层封装了复杂的 Lua 脚本，维护令牌桶的当前可用令牌数和上次补充时间。

### 用法

```java
RRateLimiter limiter = redissonClient.getRateLimiter("my:limiter");
// 初始化：每秒产生 100 个令牌，桶容量 200
limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);

// 消费一个令牌
boolean acquired = limiter.tryAcquire(1);
if (acquired) {
    // 允许通过
} else {
    // 限流
}

// 一次性消耗多个令牌
limiter.tryAcquire(10);
```

### 场景选择

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| 简单计数限流 | `Integer.increment()` + Lua | 性能最好 |
| 分布式精准限流 | `RRateLimiter` | 令牌桶算法，无需自己实现 Lua |
| 滑动窗口限流 | ZSet + Lua | 需要精确窗口边界 |
| 本地限流 | Google Guava RateLimiter | 无需网络开销 |

### 警告

当用户说"我用 Redis + Lua 自己写令牌桶/漏桶限流"时，必须直接否决。自己写容易陷入边界条件陷阱（如：并发下时间漂移、令牌计算不精确）。

---

## 布隆过滤器 RBloomFilter

### 防穿透利器

在查询 DB 前前置拦截，快速判断数据是否可能存在。

### 用法

```java
RBloomFilter<String> filter = redissonClient.getBloomFilter("my:bloom");
filter.tryInit(1_000_000, 0.01); // 预估 100 万插入，误判率 1%

// 添加
filter.add("user:123");

// 判断
boolean mightExist = filter.contains("user:123"); // 可能存在（误判）
boolean definitelyNotExist = !filter.contains("user:999"); // 一定不存在
```

### 致命避坑审查

**必须调用 `tryInit(expectedInsertions, falseProbability)`**：
- 如果实际插入量超过预估，误判率会直线上升
- 例如：预估 100 万，实际插入 500 万，误判率可能从 1% 飙升到 30%+
- 后果：正常数据被误杀（查询 DB 报不存在），用户体验受损

### 容量预估公式

```
误判率 0.01% 时，所需 bit 数 ≈ 1.2 * n（n = 元素数量）
所需 hash 函数数 = 0.7 * (bit数 / n)
```

### 替代方案

| 方案 | 适用场景 | 缺点 |
|------|----------|------|
| RBloomFilter | 大数据量防穿透 | 没法删除，误判 |
| 空值缓存 | 小数据量防穿透 | 内存消耗大 |
| 布隆过滤器 + 删除标记 | 需要删除的场景 | 实现复杂 |

---

## 延迟队列 RDelayedQueue

### 替代场景批判

当用户提出"我用 ZSet + 定时任务轮询做延迟队列"时，必须批判：
1. **轮询空扫 CPU 浪费**：每 100ms 扫一次，99.9% 是空扫
2. **延迟不精准**：轮询间隔决定延迟精度，无法做到毫秒级
3. **扩展性差**：多节点部署时，定时任务可能重复执行

### 机制说明

底层基于 `RList` 和定时任务，精准将到期元素转移到目标 Queue：
1. 元素加入延迟队列时，同时写入一个有序的 ZSet（score = 过期时间戳）
2. 后台定时任务每 100ms 检查 ZSet，找出到期元素
3. 将到期元素从 ZSet 移除，加入目标 Queue
4. 消费者从目标 Queue 消费

### 用法

```java
RQueue<String> targetQueue = redissonClient.getQueue("target:queue");
RDelayedQueue<String> delayedQueue = redissonClient.getDelayedQueue(targetQueue);

// 添加延迟元素：10 秒后到达 targetQueue
delayedQueue.offer("task", 10, TimeUnit.SECONDS);

// 取消延迟（从 ZSet 移除）
delayedQueue.remove("task");

// 销毁延迟队列
delayedQueue.destroy();
```

### 适用场景

- 订单超时取消
- 延迟通知（如：下单后 30 分钟未支付，发送提醒）
- 任务重试（失败后延迟 N 秒重试）

---

## 分布式集合增强 RMapCache

### 适用场景

支持针对每个单独的 Field 设置独立的过期时间，适合需要差异化 TTL 的场景。

### 替代场景

当用户为了实现不同字段不同过期时间，去拼凑一堆带有不同 TTL 的 String Key 时，推荐使用 `RMapCache`：

```java
// 错误：多个 String Key 实现不同 TTL
redisTemplate.opsForValue().set("user:123:name", "John", 5, TimeUnit.MINUTES);
redisTemplate.opsForValue().set("user:123:age", "30", 1, TimeUnit.HOURS);
redisTemplate.opsForValue().set("user:123:avatar", "url", 24, TimeUnit.HOURS);

// 正确：使用 RMapCache
RMapCache<String, String> mapCache = redissonClient.getMapCache("user:123");
mapCache.put("name", "John", 5, TimeUnit.MINUTES);
mapCache.put("age", "30", 1, TimeUnit.HOURS);
mapCache.put("avatar", "url", 24, TimeUnit.HOURS);
```

### 注意

`RMapCache` 底层也是 Hash，但提供了独立的 TTL 管理。在获取整体对象时依然需要 `getAll()`，性能比多个 String Key 更好。