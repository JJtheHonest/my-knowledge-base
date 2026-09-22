---
title: Reducer 与并行结果顺序
type: guide
permalink: my-kb/人工智能/lang-chain/lang-graph/reducer-与并行结果顺序
tags:
  - LangGraph
  - reducer
  - Annotated
  - concurrency
  - ordering
created: 2026-09-22
source: LangGraph 官方文档与学习讨论
---

# Reducer 与并行结果顺序

## Reducer 是什么

在 LangGraph 中，State 字段可以通过 `Annotated` 指定 reducer：

```python
completed_sections: Annotated[list[str], operator.add]
```

reducer 是一个二元函数：

```python
def reducer(current_value, new_update):
    return merged_value
```

其中：

- `current_value`：该 channel 当前已经累计的值。
- `new_update`：某个节点刚返回的更新。
- 返回值：应用更新后的新值。

## 默认更新方式：覆盖

没有 reducer 时，字段默认采用覆盖语义：

```python
class State(TypedDict):
    status: str
```

```text
running -> completed
```

最终值是 `completed`。如果多个并行节点在同一轮同时更新没有 reducer 的同一字段，LangGraph 通常无法判断应保留哪个值，会抛出 `InvalidUpdateError`。

## 列表拼接：operator.add

```python
class State(TypedDict):
    completed_sections: Annotated[list[str], operator.add]
```

```python
["section 1"] + ["section 2"]
# ["section 1", "section 2"]
```

节点必须返回列表，而不是字符串：

```python
return {"completed_sections": [section_content]}
```

`operator.add` 也可以用于数字累加：

```python
class State(TypedDict):
    total_tokens: Annotated[int, operator.add]
```

## 集合并集

```python
def merge_sets(current: set[str], update: set[str]) -> set[str]:
    return current | update


class State(TypedDict):
    keywords: Annotated[set[str], merge_sets]
```

集合并集适合并行收集和去重。它满足结合律、交换律和幂等性，通常不依赖更新顺序。

## 字典合并

```python
def merge_dicts(
    current: dict[str, str],
    update: dict[str, str],
) -> dict[str, str]:
    return current | update


class State(TypedDict):
    completed_sections: Annotated[dict[str, str], merge_dicts]
```

如果多个 Worker 写入相同 key，后应用的值会覆盖之前的值。需要严格防止冲突时：

```python
def merge_unique_dicts(current, update):
    duplicates = current.keys() & update.keys()
    if duplicates:
        raise ValueError(f"Duplicate keys: {duplicates}")
    return current | update
```

## 最大值、最小值和 Top-K

保留最高置信度：

```python
class State(TypedDict):
    max_confidence: Annotated[float, max]
```

自定义 Top-K：

```python
def keep_top_five(current, update):
    combined = current + update
    return sorted(
        combined,
        key=lambda item: item["score"],
        reverse=True,
    )[:5]


class State(TypedDict):
    search_results: Annotated[list[dict], keep_top_five]
```

## 列表去重与计数合并

保留首次出现顺序并去重：

```python
def append_unique(current: list[str], update: list[str]) -> list[str]:
    return list(dict.fromkeys(current + update))
```

合并计数字典：

```python
from collections import Counter


def merge_counts(current: dict[str, int], update: dict[str, int]):
    return dict(Counter(current) + Counter(update))
```

## 消息 reducer

消息列表应优先使用 LangGraph 的 `add_messages`：

```python
class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

它不仅追加新消息，还能根据消息 ID 更新已有消息，并处理部分字典格式的消息反序列化。直接使用 `operator.add` 不具备这些消息语义。

## 绕过 reducer：Overwrite

对于累加字段，返回空值通常不能清空已有内容：

```python
errors: Annotated[list[str], operator.add]

return {"errors": []}
```

这相当于把空列表追加进去，原错误仍然存在。需要完全替换时使用：

```python
from langgraph.types import Overwrite

return {"errors": Overwrite([])}
```

`Overwrite` 会绕过该字段的 reducer，直接设置新值。

## 并行结果的顺序

`operator.add` 的列表拼接不满足交换律：

```python
["a"] + ["b"] != ["b"] + ["a"]
```

因此，业务逻辑不应依赖并行任务的网络完成顺序或模型响应耗时。

在实际运行中，章节结果常常显示为与 `Send` 列表相同的顺序。这通常不是因为 Worker 恰好按章节顺序完成，而是 LangGraph 对任务更新采用了框架定义的应用顺序。这个实现细节不应被当作业务排序保证。

如果章节顺序重要，应显式携带索引：

```python
class CompletedSection(TypedDict):
    index: int
    content: str
```

```python
def assign_workers(state: State):
    return [
        Send(
            "llm_call",
            {"index": index, "section": section},
        )
        for index, section in enumerate(state["sections"])
    ]
```

合成时排序：

```python
ordered_sections = sorted(
    state["completed_sections"],
    key=lambda item: item["index"],
)
```

这样排序语义由业务数据显式表达，不依赖并发完成时间、任务调度细节或框架内部实现。

## 并行 reducer 的设计原则

自定义 reducer 最好满足：

- 结合律：`merge(merge(a, b), c) == merge(a, merge(b, c))`
- 尽量满足交换律：`merge(a, b) == merge(b, a)`
- 必要时满足幂等性：重复应用同一更新不会改变结果

集合并集、`max`、`min` 和数字加法适合并行聚合。需要保序的列表应携带索引并在消费前排序。

## Observations

- [fact] `Annotated[field_type, reducer]` 用于声明 LangGraph channel 的更新合并规则。
- [fact] 没有 reducer 时，默认更新会覆盖旧值。
- [fact] `operator.add` 可用于列表拼接和数字累加，但列表结果的业务顺序需要谨慎处理。
- [fact] `add_messages` 适合带 ID 和序列化需求的 LangChain 消息列表。
- [fact] `Overwrite` 可以绕过 reducer 直接替换字段。
- [insight] 并行聚合的 reducer 应尽量具有结合律和交换律；否则必须显式定义顺序语义。
- [decision] 报告章节使用索引排序，不把并行更新的应用顺序当作业务保证。

## Relations

- relates_to [[WorkerState、Send 与多 Schema 节点]]
- relates_to [[super-step、状态快照与并行可见性]]
- relates_to [[LangGraph State：TypedDict 与 Pydantic 的选择]]