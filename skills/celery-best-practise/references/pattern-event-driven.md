# 模式 8：事件驱动（解耦）

**场景**：用户注册后积分/风控/营销多系统响应

## 核心配置

```python
task_default_exchange = "events"
task_default_exchange_type = "topic"
task_routes = {
    "users.signals.*": {"queue": "user_events", "routing_key": "user.signals"},
    "payments.signals.*": {"queue": "payment_events"},
}
task_create_missing_queues = True
```

## 示例：事件发布

```python
@shared_task(name="users.signals.user_registered")
def user_registered_event(user_id: int, user_data: dict):
    """只做事件载体，不做下游业务"""
    return {
        "event_type": "user.registered",
        "user_id": user_id,
        "username": user_data.get("username"),
    }
```

## 示例：事件消费（积分系统）

```python
@shared_task(name="points.handlers.on_user_registered", acks_late=True, max_retries=2)
def award_welcome_points(self, user_id: int, user_data: dict):
    _create_points_record(user_id, 100, "注册奖励")
```

## Fanout 广播（一个事件多系统消费）

```python
task_routes = {
    "orders.signals.*": {
        "exchange": "order_fanout",
        "exchange_type": "fanout",
        "routing_key": "",
    },
}
# RabbitMQ 手动绑定多个队列到同一 Exchange
```

## 防坑

- 下游消费者必须幂等（acks_late 可能重复投递）
- 事件格式必须统一（JSON Schema）
- 新增消费者不改发布者（Fanout 优势）
