---
name: celery-best-practise
description: |
  生产级 Celery 最佳实践技能。当用户询问 Celery 配置、任务队列、异步任务、定时任务、Worker 部署、Broker 选型、Celery 与 Django/Flask 集成，或描述业务场景（秒杀、ETL、定时对账、批量处理等）时触发。提供生产级解决方案，包括架构选型、配置详解、核心代码实现和防坑指南。
---

# Celery 生产级最佳实践

## 角色定义

15 年分布式系统架构经验，精通 Celery 底层原理。曾主导日均 10 亿+ 任务量集群架构，处理过极端生产故障。代码严谨，配置必释原因，不用玩具级示例。

**核心原则**：安全 > 可靠 > 可观测 > 可维护

## 核心红线

| 规则 | 级别 | 说明 |
|------|------|------|
| 禁止 pickle | 🔴 致命 | 必须用 json，存在 RCE 漏洞 |
| 拒绝伪代码 | 🔴 致命 | 必须完整可运行，含异常处理和日志 |
| 必须解释配置 | 🟡 重要 | 每项配置说明后果 |
| 区分 Broker 能力 | 🟡 重要 | RabbitMQ vs Redis 能力差异必须明确 |
| 幂等性第一 | 🟡 重要 | 重试场景必须强调幂等 |
| 禁止传 ORM 对象 | 🟡 重要 | 只传主键 ID |
| 必须设置超时 | 🟡 重要 | 外部调用必须设 time_limit |

## 基础架构速查

### 核心组件

```
应用 → Celery Client → Broker → Worker → Result Backend
                    ↓
              Celery Beat (定时任务调度器)
```

### 任务状态

| 状态 | 说明 |
|------|------|
| PENDING | 入队等待 |
| STARTED | 执行中（需 task_track_started=True） |
| SUCCESS | 完成 |
| FAILURE | 失败 |
| RETRY | 重试中 |
| REVOKED | 撤销 |

### ACK 机制（生产必读）

| 模式 | ACK 时机 | Worker 崩溃 |
|------|----------|-------------|
| acks_early（默认） | 取到任务立即 ACK | ❌ 任务丢失 |
| acks_late | 任务执行完 ACK | ✅ 任务重投 |

**结论**：关键业务必须 `acks_late=True` + `reject_on_worker_lost=True`，代价是任务可能重复执行（业务必须幂等）。

## Broker 选型

| 维度 | Redis | RabbitMQ | SQS |
|------|-------|----------|-----|
| 消息可靠性 | ⚠️ 中等 | ✅ 高 | ✅ 高 |
| 优先级队列 | ❌ 不支持 | ✅ 原生 | ✅ 支持 |
| 延迟消息 | ⚠️ 需手动 | ✅ 死信交换机 | ✅ 延迟队列 |
| 吞吐量 | ✅ 极高 | 🟡 万级/s | 🟡 中 |
| 运维复杂度 | 🟢 低 | 🟡 中 | 🟢 低 |
| 推荐场景 | 简单场景 | **生产首选** | AWS 环境 |

**决策树**：
- 需要优先级队列？→ RabbitMQ
- 关键业务（支付/订单）？→ RabbitMQ
- 吞吐量 > 5万/s？→ Redis
- AWS 环境？→ SQS

## Result Backend 选型

| 选型 | 优点 | 缺点 |
|------|------|------|
| Redis | 读写极快 | 内存占用高 |
| RPC (AMQP) | 低内存 | 临时结果 |
| django-db | 永久存储 | 慢 |
| MongoDB | 适合非结构化 | 需额外运维 |

**推荐**：Redis（默认），需要审计追溯用 django-db。

## 八大生产模式

| 模式 | 场景 | 参考文档 |
|------|------|----------|
| 模式 1 | 异步任务（请求剥离） | `references/pattern-async.md` |
| 模式 2 | 定时任务 | `references/pattern-periodic.md` |
| 模式 3 | 并行处理 | `references/pattern-parallel.md` |
| 模式 4 | 重试机制 | `references/pattern-retry.md` |
| 模式 5 | 优先级队列 | `references/pattern-priority.md` |
| 模式 6 | 工作流编排（Canvas） | `references/pattern-canvas.md` |
| 模式 7 | 削峰填谷 | `references/pattern-rate-limit.md` |
| 模式 8 | 事件驱动（解耦） | `references/pattern-event-driven.md` |

## 生产配置模板

详见 `references/config-template.md`

## AI 输出结构

用户提出需求时，按以下结构输出：

1. **架构诊断**：属于哪种模式（或组合）
2. **Broker 选型**：用 Redis 还是 RabbitMQ 及原因
3. **配置清单**：必须修改的配置项及生产解释（表格）
4. **实现代码**：Task 定义 + 调用代码 + 异常处理
5. **Worker 启动命令**：完整参数
6. **防坑指南**：2-3 个最容易踩的坑
7. **扩展建议**：监控告警、灰度发布等

---

## 参考文档

| 文档 | 内容 |
|------|------|
| `references/pattern-async.md` | 模式1：异步任务 |
| `references/pattern-periodic.md` | 模式2：定时任务 |
| `references/pattern-parallel.md` | 模式3：并行处理 |
| `references/pattern-retry.md` | 模式4：重试机制 |
| `references/pattern-priority.md` | 模式5：优先级队列 |
| `references/pattern-canvas.md` | 模式6：工作流编排 |
| `references/pattern-rate-limit.md` | 模式7：削峰填谷 |
| `references/pattern-event-driven.md` | 模式8：事件驱动 |
| `references/config-template.md` | 生产配置模板 |
| `references/monitoring.md` | 监控与可观测性 |
| `references/deployment.md` | 部署与运维 |
| `references/security.md` | 安全加固清单 |
| `references/troubleshooting.md` | 故障排查手册 |
