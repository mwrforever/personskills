# 模式 3：并行处理（多机分而治之）

**场景**：大文件处理、批量图片、数据清洗、ML 推理

## 核心配置

```python
worker_prefetch_multiplier = 1   # 关键！防止长任务饿死短任务
worker_concurrency = CPU核心数   # CPU 密集型
worker_pool = "prefork"          # CPU 密集
worker_pool = "gevent"           # IO 密集
worker_max_tasks_per_child = 1000
worker_max_memory_per_child = 200000
```

## 示例：chord 并行处理 + 汇总

```python
@shared_task(bind=True, acks_late=True, max_retries=3, soft_time_limit=600)
def process_chunk(self, page_num: int, chunk_size: int) -> dict:
    items = list(MyModel.objects.all().order_by("id")[start:end].values("id", "field1"))
    for item in items:
        if _is_processed(item["id"]):
            continue
        _do_heavy_computation(item)
        _mark_processed(item["id"])
    return {"page": page_num, "success": count}

@shared_task(name="merge_results")
def merge_results(results: list, total_items: int) -> dict:
    total = sum(r["success"] for r in results)
    return {"total_success": total, "total_items": total_items}

def start_parallel_processing(queryset, chunk_size: int = 500):
    paginator = Paginator(queryset, chunk_size)
    header = [_process_chunk.s(i, chunk_size) for i in paginator.page_range]
    result = chord(header)(_merge_results.s(total_items=paginator.count))
    return result
```

## gevent Worker 启动（IO 密集型）

```bash
celery -A config.celery worker -P gevent -c 500 --prefetch-multiplier=1 -Q io_tasks
```

## 防坑

- prefetch_multiplier 默认值 4 是性能杀手
- gevent 必须 monkey.patch_all()
- 数据库连接池 >= concurrency
