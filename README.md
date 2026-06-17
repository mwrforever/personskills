# Personal Skills

Personal Claude Code skills for enhanced developer productivity.

## Skills

### Developer Growth Analysis
Analyzes your recent Claude Code chat history to identify coding patterns, development gaps, and areas for improvement.

### Problem Knowledge Capture
Captures problems encountered during coding for later skill creation and knowledge management.

### Best SQLAlchemy Practise
SQLAlchemy 2.0+ production-grade best practices. Covers ORM/Core, engine configuration, N+1 protection, concurrency locks, and performance tuning.

### Celery Best Practise
Production-grade Celery best practices. Covers task queues, async tasks, scheduled tasks, Worker deployment, Broker selection, and integration with Django/Flask.

### DDL Architect
Expert DDL architect for designing table structures, schema, and data models based on production-grade standards for massive data and high concurrency.

### High Concurrency Cache Design
Cache architecture design for high-concurrency systems. Covers Redis data structures, local caching (CHM/Caffeine), Redisson advanced features, and cache consistency solutions.

### High Concurrency Idempotency Design
Idempotency and concurrency control for distributed systems. Covers deduplication, distributed locks, concurrent risk identification, and production-grade architecture.

### MyBatis-Plus Best Practise
MyBatis-Plus 3.5.4+ chain API and XML mapping best practices. Lambda chain API for single-table CRUD, XML for complex multi-table operations.

### Post Task Checklist Verifier
用于在任务清单完成后验证当前分支代码变更是否完整落地，按功能清单任务项协调子 agent 并行端到端检查实现正确性、明显业务 bug、实现级优化问题和测试缺口。

### Code Analyse Spec
代码仓库问题分析与解决方案规格化技能。默认扫描当前分支 diff，按 bug/性能/设计三维度并行派发子 agent 发现问题；与用户逐项沟通原因、风险与是否处理；为每个问题给出 2-3 种方案并标注最推荐；落盘到 `docs/analyse/{yyyy-mm-dd}-{≤10字简要描述}.md`；用子 agent 按问题项端到端自检，用户审批通过后调用 `superpowers:using-superpowers` 衔接实现。

## Installation

In Claude Code, install this plugin:

```
/plugin install personskills@mwrforever
```

Or add as marketplace:
```
/plugin marketplace add mwrforever/personskills
```
