---
title: super-step、状态快照与并行可见性
type: guide
permalink: my-kb/人工智能/lang-chain/lang-graph/super-step-、状态快照与并行可见性
tags:
  - LangGraph
  - super-step
  - snapshot
  - concurrency
  - Send
created: 2026-09-22
source: LangGraph 官方文档与学习讨论
---

# super-step、状态快照与并行可见性

## 核心结论

同一个 super-step 中并行执行的节点，读取的是该轮开始时可见的状态快照，以及各自的 `Send` 任务输入。某个节点先完成后，它返回的更新不会立即暴露给仍在执行的并行节点。

更新会先被收集，在 super-step 边界统一应用 reducer。下一轮执行的节点才能看到合并后的状态。

## 并行执行模型

```text
super-step N 的状态快照
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
 Worker 1 Worker 2 Worker 3
    │     │     │
    └─────┼─────┘
          ▼
  reducer 合并所有更新
          │
super-step N+1 的状态
```

假设三个 Worker 都返回：

```python
{"completed_sections": [content]}
```

即使 Worker 1 很快结束，它的结果也不会让 Worker 2 在当前函数执行过程中读取到。Worker 2 仍使用本轮开始时的输入。

## 对报告 Demo 的含义

`llm_call` 节点虽然在 `WorkerState` 中声明了 `completed_sections`，但它的函数体只读取：

```python
state["section"]
```

因此 Worker 之间不会互相读取已完成章节。它们只负责产生自己的章节，然后把结果写入共享 channel。

`synthesizer` 位于并行 Worker 之后：

```python
builder.add_edge("llm_call", "synthesizer")
```

它会在这一批 `llm_call` 任务的更新完成并聚合后执行，因此能读取完整的 `completed_sections`。

## 节点更新的正确写法

节点应返回更新，而不是直接修改传入的 state：

```python
def worker(state: WorkerState):
    result = generate(state["section"])
    return {"completed_sections": [result]}
```

不要依赖以下模式：

```python
def worker(state: WorkerState):
    state["completed_sections"].append(result)
    return state
```

直接修改会破坏 LangGraph 对更新、checkpoint、并行合并和重试的管理，也容易引入共享可变对象问题。

## 什么时候能看到更新

一个节点可以看到其他节点更新的典型条件是：

- 它位于后续 super-step。
- 它通过图的边在前一批节点完成后被触发。
- 前一批更新已经通过 reducer 写回状态。

它不能通过普通 State 读取看到同一 super-step 中另一个并行节点的中间结果。如果业务需要“一个任务完成后立即触发另一个任务”，应把依赖关系建模成边，而不是依赖并行节点之间的共享内存。

## 顺序执行与并行执行的区别

顺序节点之间：

```text
node_a -> node_b
```

`node_b` 可以读取 `node_a` 在前一轮产生的更新。

并行节点之间：

```text
node_a ─┐
node_b ─┼─> node_c
node_d ─┘
```

`node_a`、`node_b`、`node_d` 彼此不能读取本轮的即时更新；`node_c` 可以读取它们聚合后的结果。

## Observations

- [fact] LangGraph 使用 super-step 和边界同步处理并行状态更新。
- [fact] 同一轮并行节点不会实时共享彼此的返回值。
- [fact] 下一阶段节点可以读取上一阶段经过 reducer 合并后的结果。
- [insight] LangGraph 的 State 更接近消息传递和版本化快照，而不是多个节点共同读写的线程共享对象。
- [decision] 需要中间结果依赖时，应使用图边拆分执行阶段，而不是让并行 Worker 互相轮询 State。

## Relations

- relates_to [[WorkerState、Send 与多 Schema 节点]]
- relates_to [[Reducer 与并行结果顺序]]