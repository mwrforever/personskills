# 监控与可观测性

## 监控体系架构

```
监控数据采集层
├── Flower (实时面板)
├── Events (任务事件)
└── 自定义 Metrics (业务指标)
        ↓
监控存储层: Prometheus / Grafana / ELK / Datadog
        ↓
告警层: 钉钉 / 飞书 / PagerDuty / 短信 / 邮件
```

## Flower 部署

```bash
# 安装
pip install flower

# 启动
celery -A config.celery flower --port=5555

# 生产配置
celery -A config.celery flower \
    --port=5555 \
    --url_prefix=/celery-flower \
    --basic_auth=admin:your_password \
    --max_tasks=10000 \
    --purge_offline_workers=300 \
    --persistent=True
```

## 自定义 Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

TASK_TOTAL = Counter("celery_task_total", "任务总数", ["task_name", "status"])
TASK_DURATION = Histogram("celery_task_duration_seconds", "执行时长", ["task_name"],
    buckets=[0.1, 0.5, 1, 2, 5, 10, 30, 60, 120, 300, 600])
TASK_QUEUE_DEPTH = Gauge("celery_queue_depth", "队列深度", ["queue_name"])
TASK_RETRIES = Counter("celery_task_retries_total", "重试次数", ["task_name"])

def track_metrics(task_func):
    @wraps(task_func)
    def wrapper(self, *args, **kwargs):
        start = time.time()
        try:
            result = task_func(self, *args, **kwargs)
            TASK_TOTAL.labels(task_name=task_func.name, status="success").inc()
            return result
        except self.Retry:
            TASK_RETRIES.labels(task_name=task_func.name).inc()
            raise
        except:
            TASK_TOTAL.labels(task_name=task_func.name, status="failure").inc()
            raise
        finally:
            TASK_DURATION.labels(task_name=task_func.name).observe(time.time() - start)
    return wrapper

@shared_task
@track_metrics
def process_order(self, order_id: str):
    pass
```

## 关键告警规则

| 告警项 | 条件 | 级别 |
|--------|------|------|
| 队列积压 | > 10000 持续 5 分钟 | P1 |
| Worker 全离线 | 所有 Worker 离线 | P0 |
| 任务失败率 | 最近 10 分钟 > 10% | P2 |
| 执行时间飙升 | P99 > 平时 3 倍 | P2 |
| 重试次数飙升 | 最近 10 分钟 > 100 | P2 |
| Worker 内存 | > 80% | P2 |
| Beat 离线 | 进程不在运行 | P1 |
