---
title: WorkerState、Send 与多 Schema 节点
type: guide
permalink: my-kb/人工智能/lang-chain/lang-graph/worker-state-、-send-与多-schema-节点
tags:
  - LangGraph
  - Send
  - WorkerState
  - State
created: 2026-09-22
source: LangGraph 官方文档与学习讨论
---

# WorkerState、Send 与多 Schema 节点

## 核心结论

LangGraph 中一个 Graph 可以使用多个 State schema，但这不表示存在多份独立的全局状态。通常有三类 schema：

- 主 State：定义 Graph 的共享状态 channel、reducer，以及默认的输入/输出边界。
- 节点输入 schema：描述某个节点需要读取的状态视图，可以是主 State 的子集。
- 私有或任务 schema：描述节点之间传递的局部数据，或 `Send` 创建的单次任务输入。

因此，`WorkerState` 不是第二份完整的全局 State，而是 Worker 节点的局部输入接口。

## 示例

```python
class State(TypedDict):
    topic: str
    sections: list[Section]
    completed_sections: Annotated[list[str], operator.add]
    final_report: str


class WorkerState(TypedDict):
    section: Section
    completed_sections: Annotated[list[str], operator.add]
```

```python
builder = StateGraph(State)
builder.add_node("orchestrator", orchestrator)
builder.add_node("llm_call", llm_call)
```

节点函数的注解告诉 LangGraph 这个节点的输入形状：

```python
def llm_call(state: WorkerState):
    ...
```

## WorkerState 是否必须是主 State 的子集

不必须。它可以：

- 只声明主 State 中的一部分字段。
- 声明主 State 中不存在的私有字段。
- 声明由 `Send` 为单次任务提供的局部字段。

例如：

```python
class WorkerState(TypedDict):
    section: Section
```

主 State 没有 `section`，但 `Send` 可以为每个 Worker 注入它：

```python
def assign_workers(state: State):
    return [
        Send("llm_call", {"section": section})
        for section in state["sections"]
    ]
```

`section` 属于当前 Worker 任务的局部输入，不会自动成为共享 State channel。

如果节点返回的字段需要进入主图、被后续节点读取或被 reducer 聚合，则应在图可识别的 State schema 中声明。示例中的 `completed_sections` 同名出现在主 State 和 WorkerState 中，因此 Worker 的返回值可以写入主图的共享 channel。

## Send 的运行过程

假设规划器生成三个章节：

```python
sections = [section_1, section_2, section_3]
```

`assign_workers` 返回：

```python
[
    Send("llm_call", {"section": section_1}),
    Send("llm_call", {"section": section_2}),
    Send("llm_call", {"section": section_3}),
]
```

LangGraph 创建三个独立的 `llm_call` 任务：

```text
Worker 1: {section: section_1}
Worker 2: {section: section_2}
Worker 3: {section: section_3}
```

每个 Worker 返回局部更新：

```python
{"completed_sections": [generated_content]}
```

这些更新不会直接互相覆盖，而是通过主 State 中 `completed_sections` 的 reducer 聚合。

## 推荐的类型拆分

如果 Worker 只需要读取任务输入，可以将输入和输出分开表达：

```python
class WorkerInput(TypedDict):
    section: Section


class WorkerOutput(TypedDict):
    completed_sections: list[str]


def llm_call(state: WorkerInput) -> WorkerOutput:
    ...
```

不过，实际能否写入主图 channel，仍然取决于主 State 是否声明了相应字段和 reducer。

## 适用模式

这种设计适合：

- Orchestrator-Worker
- Map-Reduce
- 动态并行任务
- 子图或私有 channel
- 让节点只看到完成工作所需的最小数据

## 关键理解

```text
主 State:        图的共享状态和聚合 channel
WorkerState:     单个 Worker 的读取视图/任务输入
Send payload:    当前任务的局部输入
Reducer:         多个 Worker 更新写回共享 channel 时的合并规则
```

## Observations

- [fact] 节点输入 schema 可以不同于 Graph 初始化时使用的主 State schema。
- [fact] `Send` 可以把主 State 中不存在的局部字段传给目标节点。
- [insight] 多 schema 表示不同节点的状态视图和输入接口，不等于多份独立全局状态。
- [decision] Worker 节点应只声明完成任务所需的最小输入字段。

## Relations

- relates_to [[LangGraph State：TypedDict 与 Pydantic 的选择]]
- relates_to [[super-step、状态快照与并行可见性]]
- relates_to [[Reducer 与并行结果顺序]]