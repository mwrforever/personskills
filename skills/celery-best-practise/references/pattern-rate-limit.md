# 模式 7：削峰填谷

**场景**：秒杀下单、短信爆发、Webhook 回调

## 核心配置

```python
task_annotations = {
    "orders.tasks.handle_seckill": {"rate_limit": "100/s", "time_limit": 30},
    "notifications.tasks.send_sms": {"rate_limit": "50/s"},
}
worker_prefetch_multiplier = 1
```

## 示例：秒杀任务

```python
@shared_task(bind=True, acks_late=True, max_retries=2)
def handle_seckill_order(self, user_id: str, product_id: str):
    redis = get_redis_connection()
    # 幂等检查
    if not redis.set(f"seckill:{user_id}:{product_id}", "1", nx=True, ex=300):
        return {"status": "duplicate"}
    
    with transaction.atomic():
        product = Product.objects.select_for_update().get(id=product_id)
        if product.stock <= 0:
            return {"status": "sold_out"}
        product.stock -= 1
        product.save()
        Order.objects.create(user_id=user_id, product_id=product_id)
    return {"status": "success"}
```

## 防坑

- rate_limit 格式必须是 "100/s" 或 "5000/m"
- Redis 队列需监控长度防止 OOM
- 接口层极轻量，所有重逻辑在 Worker
