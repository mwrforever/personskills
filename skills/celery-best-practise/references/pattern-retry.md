# 模式 4：重试机制

**场景**：第三方 API 调用、支付网关、瞬时故障容忍

## 异常分类

| 异常类型 | 举例 | 重试？ |
|----------|------|--------|
| 网络超时/连接失败 | Timeout, ConnectionError | ✅ 指数退避 |
| 服务端错误 | HTTP 500/502/503 | ✅ 指数退避 |
| 限流 | HTTP 429 | ✅ 用 Retry-After |
| 业务错误 | HTTP 400/401/404 | ❌ 不重试 |
| 资源不足 | 内存/磁盘满 | ⚠️ 谨慎 |

## 示例：第三方支付调用

```python
@shared_task(bind=True, max_retries=5, default_retry_delay=60,
             acks_late=True, reject_on_worker_lost=True)
def call_payment_gateway(self, order_id: str, payload: dict) -> dict:
    try:
        response = requests.post(url, json=payload, timeout=10)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.Timeout:
        raise self.retry(exc=exc, countdown=_get_delay(self.request.retries))
    except requests.exceptions.HTTPError as exc:
        if exc.response.status_code == 429:
            raise self.retry(exc=exc, countdown=int(exc.response.headers.get("Retry-After", 60)))
        if exc.response.status_code >= 500:
            raise self.retry(exc=exc, countdown=_get_delay(self.request.retries))
        raise PaymentBusinessError(f"HTTP {exc.response.status_code}")  # 不重试

def _get_delay(retry_count: int) -> int:
    delay = 60 * (2 ** retry_count)
    return delay + random.randint(0, delay // 2)  # 指数退避 + 抖动
```

## 重试耗尽告警

```python
@signals.task_failure.connect
def task_failure_handler(sender, task_id, exception, **kw):
    if isinstance(exception, PaymentBusinessError):
        return  # 业务错误不告警
    _send_alert(f"Task {sender.name}[{task_id}] failed after retries")
```

## 防坑

- 指数退避 + 随机抖动防止重试风暴
- 重试耗尽必须告警
- 支付接口必须传幂等键
