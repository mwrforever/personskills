# 模式 2：定时任务

**场景**：Session 清理、财务对账、数据同步、订单关闭

## 核心配置

```python
beat_scheduler = "django_celery_beat.schedulers:DatabaseScheduler"  # 生产必须用 DB 调度器
timezone = "Asia/Shanghai"
enable_utc = True
```

## 示例：Beat 调度配置

```python
app.conf.beat_schedule = {
    "clear-expired-sessions": {
        "task": "maintenance.tasks.clear_expired_sessions",
        "schedule": crontab(hour=0, minute=30),
        "options": {"queue": "maintenance"},
    },
    "reconciliation-daily": {
        "task": "finance.tasks.daily_reconciliation",
        "schedule": crontab(hour=1, minute=0),
        "options": {"queue": "finance", "time_limit": 3600},
    },
}
```

## 示例：带分布式锁的定时任务

```python
@shared_task(bind=True, acks_late=True, max_retries=2, soft_time_limit=300)
def clear_expired_sessions(self, batch_size: int = 5000):
    lock_key = "lock:clear_expired_sessions"
    if not cache.add(lock_key, "1", timeout=600):
        return  # 已有实例在跑，跳过
    
    try:
        cutoff = timezone.now()
        while True:
            with transaction.atomic():
                expired_ids = list(
                    Session.objects.filter(expire_date__lt=cutoff)
                    .values_list("session_key", flat=True)[:batch_size]
                )
                if not expired_ids:
                    break
                Session.objects.filter(session_key__in=expired_ids).delete()
    finally:
        cache.delete(lock_key)
```

## 防坑

- 只部署一个 Beat 实例（或用 django_celery_beat）
- crontab 必须设置 timezone
- 关键任务用分布式锁防重叠
