# Celery 生产配置模板

## Broker & Backend

```python
CELERY_BROKER_URL = "amqp://user:pass@rabbitmq:5672/vhost"  # 生产首选 RabbitMQ
CELERY_RESULT_BACKEND = "redis://redis:6379/1"
```

## 序列化

```python
CELERY_TASK_SERIALIZER = "json"  # 禁止 pickle！
CELERY_RESULT_SERIALIZER = "json"
CELERY_ACCEPT_CONTENT = ["json"]
```

## 时区

```python
CELERY_TIMEZONE = "Asia/Shanghai"
CELERY_ENABLE_UTC = True
```

## 任务可靠性

```python
CELERY_TASK_ACKS_LATE = True
CELERY_TASK_REJECT_ON_WORKER_LOST = True
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_DEFAULT_MAX_RETRIES = 3
CELERY_TASK_DEFAULT_RETRY_DELAY = 60
```

## Worker 性能

```python
CELERY_WORKER_PREFETCH_MULTIPLIER = 1
CELERY_WORKER_MAX_TASKS_PER_CHILD = 1000
CELERY_WORKER_MAX_MEMORY_PER_CHILD = 200000
```

## 结果

```python
CELERY_RESULT_EXPIRES = 3600
CELERY_RESULT_EXTENDED = True
```

## 优先级

```python
CELERY_TASK_QUEUE_MAX_PRIORITY = 10
CELERY_TASK_DEFAULT_PRIORITY = 5
CELERY_TASK_DEFAULT_QUEUE = "default"
```

## Beat（定时任务）

```python
CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers:DatabaseScheduler"
```

## 路由

```python
CELERY_TASK_ROUTES = {
    "orders.tasks.*": {"queue": "orders"},
    "payments.tasks.*": {"queue": "payments"},
}
CELERY_TASK_CREATE_MISSING_QUEUES = True
```

## 安全

```python
CELERY_TASK_ALWAYS_EAGER = False
CELERY_WORKER_SEND_TASK_EVENTS = True
CELERY_TASK_SEND_SENT_EVENT = True
CELERY_BROKER_CONNECTION_MAX_RETRIES = 0
```
