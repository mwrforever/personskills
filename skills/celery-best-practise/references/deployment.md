# 部署与运维

## Supervisor 配置

```ini
; Worker 配置
[program:celery-worker-default]
command=/opt/venv/bin/celery -A config.celery worker -Q default -c 8 --prefetch-multiplier=1 --max-tasks-per-child=1000 --max-memory-per-child=200000 -n default_worker@%%h -l info -f /var/log/celery/worker-default.log
directory=/opt/app
user=celery
autostart=true
autorestart=true
stopwaitsecs=600
stopasgroup=true
killasgroup=true

; Beat 配置
[program:celery-beat]
command=/opt/venv/bin/celery -A config.celery beat -S django -l info --pidfile=/var/run/celery/beat.pid
directory=/opt/app
user=celery
autostart=true
autorestart=true

; Flower 监控
[program:celery-flower]
command=/opt/venv/bin/celery -A config.celery flower --port=5555 --basic_auth=admin:pass
directory=/opt/app
autostart=true
autorestart=true
```

## 零停机部署脚本

```bash
#!/bin/bash
set -e
supervisorctl stop celery-worker-default celery-worker-orders
sleep 5
while pgrep -f "celery worker" > /dev/null; do sleep 2; done
git pull origin main && pip install -r requirements.txt && python manage.py migrate
supervisorctl start celery-worker-default celery-worker-orders celery-beat
sleep 10 && celery -A config.celery inspect active_queues
```

## 日志配置

```python
LOGGING = {
    "formatters": {
        "celery": {
            "format": "%(asctime)s | %(levelname)-8s | [%(task_name)s:%(task_id)s] | %(message)s",
        },
    },
    "handlers": {
        "celery_file": {
            "level": "INFO",
            "class": "RotatingFileHandler",
            "filename": "/var/log/celery/tasks.log",
            "maxBytes": 100 * 1024 * 1024,
            "backupCount": 10,
        },
    },
    "loggers": {
        "celery": {"handlers": ["celery_file"], "level": "INFO"},
    },
}
```

## 性能调优

### Prefetch Multiplier 影响

| 设置 | 行为 | 结果 |
|------|------|------|
| prefetch=4 | Child 1: [T1,T2,T3,T4]，T1 长时 T2-T4 被锁死 | 负载不均 |
| prefetch=1 | 处理完才取下一个 | 负载均衡 |

### 调优清单

| 调优项 | 默认值 | 调优后 | 效果 |
|--------|--------|--------|------|
| prefetch_multiplier | 4 | 1 | 负载均衡 3-5x |
| max_tasks_per_child | 无限 | 1000 | 内存泄漏消除 |
| Result Backend | Django ORM | Redis | 查询 100x 提升 |
| worker_pool (IO) | prefork | gevent | 吞吐 5-10x |

### 性能瓶颈排查

```python
from celery import current_app
inspector = current_app.control.inspect()
active = inspector.active()      # 活跃任务
reserved = inspector.reserved()  # 预留任务（多说明 prefetch 太大）
```
