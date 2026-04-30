# 安全加固清单

| 安全项 | 检查点 | 配置 |
|--------|--------|------|
| 序列化安全 | 禁止 pickle | `task_serializer="json"` |
| Broker 认证 | 有密码/证书 | RabbitMQ: 用户名密码; Redis: requirepass |
| Broker 加密 | 传输链路加密 | RabbitMQ: SSL; Redis: rediss:// |
| Flower 认证 | 访问控制 | `--basic_auth=user:pass` |
| 任务参数 | 不传敏感信息 | 只传 ID，任务内部查询 |
| 结果后端 | 结果不含敏感数据 | 返回值不含密码等 |
| eager 模式 | 生产关闭 | `task_always_eager=False` |
| Worker 权限 | 非 root 运行 | 专用 celery 用户 |
| 网络隔离 | Worker 与 Broker 内网 | 不暴露公网端口 |
| 限流保护 | 防恶意刷 | 关键任务配置 rate_limit |
