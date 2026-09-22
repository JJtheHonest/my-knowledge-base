---
title: LangGraph State：TypedDict 与 Pydantic 的选择
type: guide
permalink: my-kb/人工智能/lang-chain/lang-graph/lang-graph-state-typed-dict-与-pydantic-的选择
tags:
- LangGraph
- LangChain
- TypedDict
- Pydantic
- Python
created: 2026-09-19
source: LangGraph 官方文档与学习讨论
---

# LangGraph State：TypedDict 与 Pydantic 的选择

## 结论

在 LangGraph 中，通常优先使用 `TypedDict` 声明图的 State；Pydantic 更适合用于系统边界、Tool 参数和 LLM structured output 的运行时校验。

推荐的组合方式是：

- 图内部共享状态：`TypedDict`
- 需要默认值但不需要复杂校验：`dataclass`
- 外部 API 输入、Tool 参数、LLM 结构化输出：Pydantic
- 需要保证某个节点输出合法：在该节点内显式调用 `model_validate`

一句话概括：**内部状态重结构，系统边界重校验。**

## 两者的定位

`TypedDict` 是静态类型工具，用来描述字典应包含哪些键以及每个键的类型。它不负责在运行时构造对象或校验数据。

Pydantic `BaseModel` 是运行时数据模型。它可以解析、转换和校验输入，并提供默认值、字段约束和自定义 validator。

因此，两者并不是简单的“低级版与高级版”关系，而是解决不同问题：

- `TypedDict` 回答“这个字典在代码中应当是什么形状”。
- Pydantic 回答“运行时收到的数据是否有效，以及如何解析为规范对象”。

## 对比表

| 维度 | `TypedDict` | Pydantic `BaseModel` |
|---|---|---|
| 主要目的 | 静态描述字典结构 | 运行时解析与校验数据 |
| 静态类型检查 | 支持 | 支持 |
| 运行时校验 | 不支持 | 支持 |
| 数据访问 | `state["field"]` | `state.field` |
| 默认值 | 不直接提供 | 原生支持 |
| 字段约束与 validator | 不支持 | 支持 |
| 类型转换 | 不支持 | 支持，但需注意隐式 coercion |
| 运行时开销 | 很低 | 有对象构造和递归校验成本 |
| 序列化结果 | 天然是普通字典 | 通常需要 `model_dump()` |
| LangGraph reducer | 可通过 `Annotated` 自然声明 | 可以使用，但整体收益常受校验限制影响 |
| LangGraph 官方定位 | 主要、默认方式 | 特定场景可选 |

## 为什么 LangGraph 官方示例偏好 TypedDict

### State 是共享状态通道，不只是输入模型

LangGraph 的 State 主要描述图中有哪些状态字段，以及各字段更新时如何合并。节点通常只返回局部更新，而不是重新构造完整对象：

```python
class State(TypedDict):
    email_content: str
    classification: EmailClassification | None
    draft_response: str | None


def classify_email(state: State):
    return {
        "classification": {
            "intent": "billing",
            "urgency": "high",
            "topic": "refund",
            "summary": "Customer requests a refund",
        }
    }
```

LangGraph 将节点返回的局部字典合并进完整状态。这个过程更接近“更新状态通道”，而不是“反复创建领域模型实例”。

### TypedDict 足以表达 reducer

State 字段不仅有类型，还可能带有 reducer，用于控制多个更新如何合并：

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    topic: str
```

`messages` 的 `Annotated` 元数据告诉 LangGraph 使用 `add_messages` 合并更新。对于“字段 + reducer”的状态定义，`TypedDict` 简洁且直接。

### Pydantic State 不是逐节点完整校验

根据当前 LangGraph 官方文档，使用 Pydantic 作为 State schema 时存在以下限制：

- 运行时校验只发生在进入第一个节点的输入上，而不是每个后续节点及其输出上。
- 图的最终输出不是 Pydantic model 实例，而是普通字典。
- Pydantic 校验错误不一定能指出错误来自哪个节点。
- 递归校验可能慢于 `TypedDict` 或 `dataclass`。

因此，将 State 改为 Pydantic，并不等价于在每次状态更新后执行：

```python
State.model_validate(current_state)
```

这使得 Pydantic 对 LangGraph 内部 State 的核心优势有所减弱。当前更高层的 LangChain `create_agent` 也不支持使用 Pydantic state schema。

## 为什么 LangChain 示例中经常使用 Pydantic

LangChain 中的 Pydantic 通常用于以下位置，而不一定用于完整 Agent State：

- Tool 的参数 schema
- LLM structured output
- HTTP/API 请求和响应
- 用户输入校验
- 配置和领域对象

这些数据来自模型、用户或外部系统，属于不可信边界，运行时校验很有价值。例如：

```python
from typing import Literal
from pydantic import BaseModel, Field


class EmailClassification(BaseModel):
    intent: Literal["question", "bug", "billing", "feature", "complex"]
    urgency: Literal["low", "medium", "high", "critical"]
    topic: str
    summary: str = Field(description="A concise summary of the email")


classifier = model.with_structured_output(EmailClassification)
result = classifier.invoke(messages)
```

这里 Pydantic 用来约束不可预测的 LLM 输出，比用于可信的内部状态流转更有价值。

## 推荐的混合设计

外层 State 使用 `TypedDict`，复杂且需要校验的字段使用 Pydantic：

```python
from typing import Annotated, Literal, NotRequired
from typing_extensions import TypedDict
from pydantic import BaseModel
from langchain.messages import AnyMessage
from langgraph.graph.message import add_messages


class EmailClassification(BaseModel):
    intent: Literal["question", "bug", "billing", "feature", "complex"]
    urgency: Literal["low", "medium", "high", "critical"]
    topic: str
    summary: str


class EmailAgentState(TypedDict):
    email_content: str
    sender_email: str
    email_id: str
    classification: NotRequired[EmailClassification]
    search_results: NotRequired[list[str]]
    customer_history: NotRequired[dict]
    draft_response: NotRequired[str]
    messages: Annotated[list[AnyMessage], add_messages]
```

这种设计兼顾了：

- State 更新和 checkpoint 的轻量性
- reducer 的自然表达
- LLM 结构化输出的运行时校验
- IDE 和类型检查器的静态提示

## `NotRequired[T]` 与 `T | None`

这两个声明的语义不同：

```python
class StateA(TypedDict):
    classification: EmailClassification | None
```

表示 `classification` 这个键应当存在，但值可以为 `None`。

```python
class StateB(TypedDict):
    classification: NotRequired[EmailClassification]
```

表示这个键本身可以不存在。

LangGraph State 通常在节点执行过程中逐步形成，因此尚未生成的中间字段常适合使用 `NotRequired`。如果希望初始状态显式包含所有字段，则可以使用 `T | None` 并在初始化时传入 `None`。

需要注意：`TypedDict` 的 required/optional 主要约束静态类型检查，不能替代运行时验证。

## 何时直接使用 Pydantic State

可以在以下条件下考虑 Pydantic State：

- 图入口直接接收不可信的外部数据。
- 强依赖字段约束、默认值、validator 或类型转换。
- State 较小且性能不敏感。
- 可以接受图输出仍为字典。
- 可以接受后续节点更新不会自动逐步严格校验。

示例：

```python
from pydantic import BaseModel, EmailStr, Field


class EmailAgentState(BaseModel):
    email_content: str = Field(min_length=1)
    sender_email: EmailStr
    email_id: str
    classification: EmailClassification | None = None
```

如果需要严格保证每个节点的输出，则应在节点内部显式验证：

```python
class ClassificationResult(BaseModel):
    intent: Literal["question", "bug", "billing", "feature", "complex"]
    urgency: Literal["low", "medium", "high", "critical"]
    topic: str
    summary: str


def classify_email(state: EmailAgentState):
    raw_result = call_classifier(state.email_content)
    result = ClassificationResult.model_validate(raw_result)
    return {"classification": result.model_dump()}
```

这样校验失败时可以准确定位到具体节点，而不是依赖 State schema 提供隐式的全流程保障。

## 选型清单

| 场景 | 推荐方案 |
|---|---|
| LangGraph 内部共享 State | `TypedDict` |
| 消息列表和 reducer | `TypedDict` + `Annotated` |
| State 需要默认值 | `dataclass`，或明确初始化字典 |
| LLM structured output | Pydantic |
| Tool 参数 | Pydantic |
| 外部 API 输入输出 | Pydantic |
| State 中的复杂领域字段 | 外层 `TypedDict` + 内层 Pydantic |
| 每个节点都需要严格校验 | 节点内显式 `model_validate` |

## Observations

- [fact] LangGraph 支持使用 `TypedDict`、`dataclass` 或 Pydantic model 声明 State schema。
- [fact] LangGraph 官方将 `TypedDict` 作为主要文档方式；需要默认值时建议考虑 `dataclass`。
- [fact] Pydantic State 的运行时校验只覆盖进入第一个节点的输入，不覆盖后续节点更新和输出。
- [fact] 图的最终输出不会自动成为 Pydantic model 实例。
- [insight] LangGraph State 的主要职责是描述状态通道、局部更新和 reducer，而不是承担端到端的数据验证。
- [insight] Pydantic 的最高价值通常位于不可信的数据边界，例如用户输入、外部 API 和 LLM 输出。
- [decision] LangGraph 项目默认使用 `TypedDict` 定义 State，在边界和复杂字段上组合使用 Pydantic。
- [decision] 如需逐节点强校验，在节点内部显式调用 `model_validate`，不要假设 Pydantic State 会自动完成全流程验证。

## Relations

- relates_to [[super-step、状态快照与并行可见性]]
- relates_to [[WorkerState、Send 与多 Schema 节点]]
- relates_to [[Reducer 与并行结果顺序]]

## 参考资料

- [LangGraph Graph API：State schema](https://docs.langchain.com/oss/python/langgraph/graph-api#schema)
- [LangGraph：Use Pydantic models for graph state](https://docs.langchain.com/oss/python/langgraph/use-graph-api#use-pydantic-models-for-graph-state)
