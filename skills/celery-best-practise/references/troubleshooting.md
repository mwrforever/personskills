# 故障排查手册

## 常见故障

| 故障现象 | 可能原因 | 排查命令 | 解决方案 |
|----------|----------|----------|----------|
| 任务不入队 | Broker 连接断开 | `celery inspect active` | 检查 Broker 状态 |
| 任务入队不执行 | Worker 全部离线 | `celery inspect ping` | 重启 Worker |
| 任务执行结果丢失 | 未配置 Backend | 检查 result_backend | 配置 Result Backend |
| 任务重复执行 | acks_late + 无幂等 | 检查任务日志 | 实现幂等性 |
| Worker 内存飙升 | 内存泄漏 | `top -p $(pgrep -f celery)` | 设置 max_tasks_per_child |
| Worker CPU 100% | 死循环/密集计算 | `py-spy top` | 优化算法/增加超时 |
| 队列积压不减少 | 消费 < 生产 | 检查队列深度 | 增加 Worker 或限流 |
| 定时任务不执行 | Beat 挂了/多实例 | `ps aux | grep beat` | 只保留一个 Beat |
| gevent Worker 卡死 | 阻塞调用 | 检查 time.sleep | 替换为 gevent.sleep |
| prefetch 不生效 | 版本过旧 | `celery --version` | 升级到 5.x |

## Django/Flask 集成

### Django 目录结构

```
project/
├── config/
│   ├── __init__.py
│   ├── settings.py
│   └── celery.py          # Celery App 定义
├── orders/
│   ├── tasks.py
│   └── views.py
└── manage.py
```

### Django 集成

```python
# config/celery.py
from celery import Celery
app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()

# config/__init__.py
from .celery import app as celery_app
__all__ = ("celery_app",)

# settings.py
CELERY_BROKER_URL = "amqp://..."
CELERY_RESULT_BACKEND = "redis://..."
CELERY_TASK_SERIALIZER = "json"
CELERY_TASK_ACKS_LATE = True
CELERY_WORKER_PREFETCH_MULTIPLIER = 1
CELERY_TIMEZONE = "Asia/Shanghai"
CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers:DatabaseScheduler"
INSTALLED_APPS = [..., "django_celery_beat"]
```

### Flask 集成

```python
from flask import Flask
from celery import Celery

def make_celery(app: Flask) -> Celery:
    celery = Celery(app.import_name, broker=app.config["CELERY_BROKER_URL"])
    celery.conf.update(app.config)
    
    class ContextTask(celery.Task):
        def __call__(self, *args, **kwargs):
            with app.app_context():
                return self.run(*args, **kwargs)
    celery.Task = ContextTask
    return celery

app = Flask(__name__)
app.config.update(CELERY_BROKER_URL="redis://...", CELERY_TASK_SERIALIZER="json")
celery = make_celery(app)
```
