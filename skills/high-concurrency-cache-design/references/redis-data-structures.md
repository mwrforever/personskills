# Redis 全量数据结构实战与避坑矩阵

## String

**适用场景**：普通 KV、`INCR` 原子计数

**避坑**：
- 避免缓存超大对象（value 超过 10KB），导致网络 IO 飙升
- 如果需要存储大对象，考虑压缩或拆分

**典型命令**：`GET/SET/MSET/MGET/INCR/DECR/EXPIRE`

---

## Hash

**适用场景**：对象部分字段更新、购物车、用户画像

**避坑**：
- Field 数量超过 512 或单元素超过 64 字节时，会转为真实 Hashtable，内存暴增
- 大 Hash 禁止使用 `HSCAN` 遍历（游标迭代器是针对小 Hash 优化的）
- 拆 Key 方案：当 Hash 字段过多时，按业务边界拆分为多个 Key

**典型命令**：`HSET/HGET/HMSET/HMGET/HGETALL/HSCAN`

**内存优化**：`HSET key field 1` 后再 `HDEL key field` 不会立即释放内存，需要等待 rehash。生产环境避免频繁的 HSET/HDEL 交替。

---

## List

**适用场景**：时间线、消息队列（简单场景）、最新消息列表

**避坑**：
- **严禁用 `LPUSH+BRPOP` 做核心 MQ** —— 无 ACK 机制，消息会丢失
- List 没有消费者组概念，无法实现消息确认
- 适合的场景：轻量级异步任务、审计日志队列

**替代方案**：使用 Stream（支持消费者组、ACK、消息持久化）

**典型命令**：`LPUSH/RPOP/LRANGE/LEN/LTRIM`

---

## Set

**适用场景**：标签系统、共同好友、抽奖（`SRANDMEMBER`/`SPOP`）

**避坑**：
- `SINTER` 求交集操作复杂度 O(N*M)，**大 Set 禁止使用 SINTER**
- 例如：用户 A 有 100 万粉丝，用户 B 有 80 万粉丝，求交集可能需要遍历 8000 亿次
- 替代方案：使用 Redis Cluster 环境下拆分，或者降级到 DB 计算

**典型命令**：`SADD/SMEMBERS/SISMEMBER/SRANDMEMBER/SPOP/SINTER/SDIFF`

---

## ZSet

**适用场景**：排行榜（实时排名）、滑动窗口限流、权重队列

**避坑**：
- **严禁深层分页**：`ZRANGE key 0 -1` 越往后越慢，百万级数据可能需要秒级
- 正确做法：只查前 100 名，深分页降级 DB
- `ZREVRANGE` 同样有性能问题，不要用于"获取第 10001-10010 名"这种场景

**典型命令**：`ZADD/ZRANGE/ZREVRANGE/ZSCORE/ZINCRBY/ZRANK`

**实战技巧**：排行榜只维护前 1000 名，每日归档历史排行到 DB。

---

## Stream

**适用场景**：5.0+ 真正的消息队列、事件溯源、日志流处理

**避坑**：
- **必须使用消费者组（XGROUP）**，否则与 List 无异（无 ACK、会丢消息）
- `XACK` 必须在业务处理成功后调用
- 消费者组名不要重复创建，使用前先检查 `XGROUP INFO`

**典型命令**：
- `XADD/XRANGE/XREVRANGE/XLEN`
- `XGROUP CREATE/XGROUP READCOUNT/XACK`
- `XREAD/XREADGROUP`

**最佳实践**：
```lua
-- 消费者组消费模板
XREADGROUP GROUP mygroup consumer1 STREAMS mystream ">"  -- 读新消息
XREADGROUP GROUP mygroup consumer1 STREAMS mystream 0     -- 读 Pending 消息（重试）
```

---

## Bitmap

**适用场景**：签到打卡、亿级用户状态（活跃/沉睡）、实时在线统计

**避坑**：
- 极度节省内存（1 亿用户 = 12.5MB）
- 但 `BITCOUNT` 大范围计算极其耗时，只能离线计算热力图
- 不能用 Bitmap 做精确计数（适合布尔状态）

**典型命令**：`SETBIT/GETBIT/BITCOUNT/BITOP/BITPOS`

**实战技巧**：签到系统使用 `SETBIT user:{uid}:sign:{year} {day_of_year} 1`

---

## HyperLogLog

**适用场景**：UV 统计（独立访客数）、海量数据去重计数

**避坑**：
- 只有 ADD 没有删除操作
- 存在 0.81% 标准误差
- **严禁用于涉及金额或精确去重场景**（可能误差几百笔交易）
- 适合场景：DAU 统计、UV 预估

**典型命令**：`PFADD/PFCOUNT/PFMERGE`

---

## GEO

**适用场景**：附近的人、门店查询、LBS 服务

**避坑**：
- 底层实现就是 ZSet（score = geohash 编码）
- **禁止直接操作 ZSet**，否则破坏 GEO 数据结构
- 必须使用 GEO 专属命令：`GEORADIUS/GEORADIUSBYMEMBER/GEOPOS`

**典型命令**：`GEOADD/GEOPOS/GEODIST/GEORADIUS/GEORADIUSBYMEMBER`

**性能注意**：`GEORADIUS` 在百万级数据下可能需要秒级，考虑预计算热点区域。

---

## 数据结构选择决策树

```
需要存储数据时：
├── 需要精确去重？
│   ├── 是 → Set（内存允许）或 数据库唯一索引
│   └── 否 → 下一步
├── 需要有序？
│   ├── 是 → ZSet（排行榜）或 Stream（有序消息）
│   └── 否 → 下一步
├── 需要集合操作（交、并、差）？
│   ├── 是 → Set（数据量 < 10 万）否则降级 DB
│   └── 否 → 下一步
├── 需要计数器？
│   ├── 是 → String + INCR
│   └── 否 → 下一步
└── 普通 KV → String
```