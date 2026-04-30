# 模式 6：工作流编排（Canvas）

**场景**：ETL 流水线、MapReduce 汇总、多阶段处理

## 原语对比

| 原语 | 模式 | 数据流 |
|------|------|--------|
| chain | 串行 | A → B → C |
| group | 并行 | A∥B∥C |
| chord | 并行 + 回调 | (A∥B∥C) → callback |

## 核心配置

```python
result_backend = "redis://redis:6379/1"  # Canvas 必须配置
result_expires = 3600
result_extended = True
task_track_started = True
```

## 示例：ETL 流水线（chain）

```python
@shared_task(name="etl.extract")
def extract(source: str) -> dict:
    return {"source": source, "data": _fetch(source)}

@shared_task(name="etl.transform")
def transform(extracted: dict) -> dict:
    cleaned = [_clean(r) for r in extracted["data"]]
    return {"source": extracted["source"], "cleaned": cleaned}

@shared_task(name="etl.load")
def load(transformed: dict) -> dict:
    _bulk_insert(transformed["cleaned"])
    return {"loaded": len(transformed["cleaned"])}

# 启动流水线
pipeline = chain(extract.s(src), transform.s(), load.s())
pipeline.apply_async()
```

## 示例：并行 + 汇总（chord）

```python
@shared_task(name="merge_all")
def merge_all(results: list) -> dict:
    return {"total": sum(r["count"] for r in results)}

header = [process_item.s(item) for item in items]
result = chord(header)(merge_all.s())
```

## 防坑

- 未配置 result_backend 则 Canvas 报错
- chord 子任务必须设超时，否则回调永不执行
- link_error 处理链中任意步骤的失败
