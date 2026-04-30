# 模式 5：优先级队列

**场景**：VIP 用户优先处理、告警通知优先送达

**Broker 要求**：必须用 RabbitMQ（Redis 不原生支持优先级）

## 核心配置

```python
task_queue_max_priority = 10
task_default_priority = 5
task_routes = {
    "orders.tasks.process_vip": {"queue": "high_priority", "priority": 9},
    "orders.tasks.process_normal": {"queue": "default", "priority": 3},
}
```

## 示例：动态优先级

```python
process_order.apply_async(
    args=[order_id],
    priority={"vip": 9, "normal": 3, "internal": 1}.get(user_tier, 5),
    queue={"vip": "high_priority", "normal": "default"}.get(user_tier, "default"),
)
```

## Worker 分离部署（关键！）

```bash
# 高优先级 Worker
celery -A config.celery worker -Q high_priority -c 4 --prefetch-multiplier=1 -n vip@%h
# 普通 Worker
celery -A config.celery worker -Q default -c 8 --prefetch-multiplier=1 -n normal@%h
```

## 防坑

- 必须按优先级分离 Worker
- 高优队列中的任务也要短小
- `-Q` 参数顺序决定优先级
