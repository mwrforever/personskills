# 模式 1：异步任务（请求剥离）

**场景**：发邮件/短信、生成报表、文件处理、第三方 API 调用

## 核心配置

```python
task_ignore_result=True   # 发邮件等不需要结果的
task_ignore_result=False  # 生成报表等需要结果查询的
task_always_eager=False   # 生产禁止 True！
task_serializer="json"
task_time_limit=300       # 硬超时 5 分钟
task_soft_time_limit=270 # 软超时 4.5 分钟
```

## 示例：发送邮件任务（发射后不管）

```python
@shared_task(
    bind=True,
    ignore_result=True,
    max_retries=3,
    default_retry_delay=60,
    acks_late=True,
    reject_on_worker_lost=True,
    soft_time_limit=30,
    time_limit=60,
    name="email.send",
)
def send_email(self, to: list, subject: str, body: str, email_id: str = None):
    # 幂等检查
    if email_id:
        if cache.get(f"email:sent:{email_id}"):
            return
        cache.set(f"email:sent:{email_id}", "1", timeout=86400)
    
    try:
        connection = get_connection(host=EMAIL_HOST, port=EMAIL_PORT, ...)
        connection.send_messages([msg])
    except smtplib.SMTPConnectError as exc:
        raise self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))
    except smtplib.SMTPRecipientsRefused:
        raise EmailSendError("Invalid recipients")  # 业务错误不重试
```

## 示例：生成报表（需要返回结果）

```python
@shared_task(bind=True, ignore_result=False, soft_time_limit=300, time_limit=600)
def generate_report(self, report_type: str, params: dict, user_id: int) -> dict:
    tmp_path = None
    try:
        data = _query_report_data(report_type, params)
        with tempfile.NamedTemporaryFile(suffix=".xlsx", delete=False) as tmp:
            tmp_path = tmp.name
            _write_excel(data, tmp_path)
        file_url = _upload_to_storage(tmp_path)
        return {"file_url": file_url, "file_size": os.path.getsize(tmp_path)}
    finally:
        if tmp_path and os.path.exists(tmp_path):
            os.remove(tmp_path)
```

## 防坑

- 禁止传 ORM 对象，只传主键 ID
- 前端轮询状态，绝不要 `result.get(timeout=300)`
- 临时文件必须清理
