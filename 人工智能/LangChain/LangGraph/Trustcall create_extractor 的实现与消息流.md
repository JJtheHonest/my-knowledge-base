---
title: Trustcall create_extractor 的实现与消息流
tags:
- Trustcall
- LangGraph
- tool-calling
- structured-output
created: 2026-09-28
source: Trustcall GitHub 源码与对话梳理
permalink: my-kb/人工智能/lang-chain/lang-graph/trustcall-create-extractor-的实现与消息流
---

# Trustcall create_extractor 的实现与消息流

## 定位

`trustcall.create_extractor()` 返回一个 LangChain `Runnable`，用于从消息提取符合 schema 的对象，或根据新消息局部更新已有对象。它并非 LangGraph 的 `ToolNode`：普通 `ToolNode` 执行模型请求调用的业务函数；Trustcall 利用模型的 tool-calling 协议生成结构化参数，自己负责校验、修补和组装输出。源码实际上在 `create_extractor()` 中构造并编译了一个小型 `StateGraph`，返回 `coerce_inputs | compiled | filter_state`。

```text
输入规范化
  ├── existing 为空 → extract → validate ────────────┐
  └── existing 非空 → extract_updates → validate ────┤
                                      校验失败 → patch → 再校验
                                             └── filter_state → 对外结果
```

上图是概念化的主路径；源码中还有 `sync`、删除内部验证消息等节点，达到最大重试次数时会结束。`existing` 是否为真决定分支，而不是单纯检查字典键是否存在。

## 参数的实际作用

```python
from pydantic import BaseModel
from trustcall import create_extractor

class Person(BaseModel):
    name: str
    city: str

extractor = create_extractor(llm, tools=[Person], tool_choice="Person")
```

- `llm`：底层聊天模型。
- `tools=[Person]`：注册可用的输出 schema。直接提取分支用这些 schema 构造模型的函数调用描述，校验时也用它们。可以注册多种结构，例如 `[Person, Product]`。
- `tool_choice="Person"`：直接提取分支传给 `llm.bind_tools(...)`，指定使用 `Person`。默认 `None` 时由模型决定是否调用、调用哪个；`"any"` 表示要求调用某个工具而不指定具体名称。只想提取 `Person` 时通常只注册 `[Person]`，无需额外注册暂时不使用的结构。
- `messages`：本次处理的消息；也可以将字符串直接传入 `invoke()`，内部会规范化为用户消息。
- `existing`：已有对象。非空时切换到更新分支，例如 `{"Person": {"name": "小李", "city": "杭州"}}`。还支持带记录 ID 的列表格式。
- `enable_inserts`：已有对象存在时是否还允许抽取新对象，默认 `False`；更新默认开启，删除默认关闭。

注意：**更新分支并不沿用创建提取器时的 `tool_choice="Person"`**。在默认开启更新且未开启新增/删除时，`_ExtractUpdates` 给模型绑定的是内部 `PatchDoc`，并指定 `tool_choice="PatchDoc"`。

## 不传 existing：直接提取

```python
result = extractor.invoke({"messages": [("user", "我叫小李，住在杭州")]})
person = result["responses"][0]  # Person(name="小李", city="杭州")
person.model_dump()                 # {'name': '小李', 'city': '杭州'}
```

1. `_Extract` 用 `llm.bind_tools(...)` 注册 `Person` 的 JSON Schema，再调用 `bound_llm.invoke(state.messages)`。
2. 模型返回 `AIMessage(tool_calls=[{"name": "Person", "args": {"name": "小李", "city": "杭州"}, "id": "..."}])`；这是模型生成的结构化参数，不是已经执行了一个 `Person` 业务函数。
3. `validate` 根据调用名查找 schema，执行 `schema.model_validate(call["args"])`，并构造关联该 `tool_call_id` 的 `ToolMessage`。
4. `filter_state` 从最终 `AIMessage` 的 tool calls 再次校验并生成 `responses`，返回 `messages`、`responses`、`response_metadata`、`attempts`。

## 传入 existing：增量更新

```python
result = extractor.invoke({
    "messages": [("user", "我搬到上海了")],
    "existing": {"Person": {"name": "小李", "city": "杭州"}},
})
# result["responses"][0].model_dump() 预期为
# {'name': '小李', 'city': '上海'}
```

1. `_ExtractUpdates._setup` 把原对象及 schema 写入给模型的系统消息，并要求生成 JSON Patch；默认仅绑定内部 `PatchDoc` 工具。
2. 模型可能生成 `PatchDoc(json_doc_id="Person", patches=[{"op": "replace", "path": "/city", "value": "上海"}], ...)`。其中 `planned_edits` 也是 schema 要求的字段，此处省略。
3. `_teardown` 用 `_apply_patch` 在本地将 patch 应用到旧字典，生成完整的 `Person` tool call：`{"name": "小李", "city": "上海"}`。因此无需模型重写并记全所有旧字段。
4. 后续与直接提取相同：`validate` 校验，`filter_state` 返回 Pydantic 实例。`response_metadata` 对更新项可包含 `json_doc_id`。

更新不会自动写数据库或 LangGraph 长期存储；持久化由调用方负责。

## ToolMessage 究竟在哪里

确实存在，用户把它理解为“validate 的结果”是准确的。校验节点的核心代码是：

```python
output = schema.model_validate(call["args"])
return ToolMessage(
    content=output.model_dump_json(),
    name=call["name"],
    tool_call_id=call["id"],
)
```

校验失败则返回带 `status="error"` 的 `ToolMessage`，内容包含错误说明。内部流程因此类似：

```text
AIMessage:   tool_call Person(args={"name": "小李", "city": "杭州"}, id="call-1")
ToolMessage: content='{"name":"小李","city":"杭州"}', tool_call_id="call-1"
```

成功时，这条内部验证消息可被清理，不必再送模型做下一轮问答。失败时路由到 `_Patch`：错误 `ToolMessage` 用作修复上下文，模型生成 `PatchFunctionErrors`（或修正函数名），代码修改原 AI 消息的 tool call 参数，然后再校验；重试受到 `max_attempts` 限制。

**对外的 `result["messages"]` 不是完整的内部历史**：`filter_state()` 只筛选最终的 `AIMessage`，并把通过校验的参数变成 `result["responses"]`，不把内部的 `ToolMessage` 放进返回列表。达到重试上限或最终仍无有效参数时，也不能假定一定有有效的 `responses`，应检查返回内容。

## 与 ToolNode 的区别

| 对象 | 主要职责 | 典型结果 |
| --- | --- | --- |
| LangGraph `ToolNode` | 执行模型请求的真实工具函数，例如搜索或查询 | 函数执行结果的 `ToolMessage` |
| Trustcall `create_extractor` | 调模型生成 schema 参数、局部更新、校验和修补 | `responses` 中的结构化对象；内部校验也会产生 `ToolMessage` |

“没有 `ToolNode`”不等于“没有 `ToolMessage`”：消息类型与创建它的节点类型是两回事。

## 源码依据

- [`create_extractor` 签名、建图、分流与输出过滤](https://github.com/hinthornw/trustcall/blob/main/trustcall/_base.py#L127)
- [`_Extract` 调用 `bind_tools` 与模型](https://github.com/hinthornw/trustcall/blob/main/trustcall/_base.py#L609)
- [`_ExtractUpdates` 绑定 `PatchDoc`、注入旧数据与应用 patch](https://github.com/hinthornw/trustcall/blob/main/trustcall/_base.py#L653)
- [`_Patch` 错误修复分支](https://github.com/hinthornw/trustcall/blob/main/trustcall/_base.py#L1038)
- [`_ExtendedValidationNode` 生成成功或失败 `ToolMessage`](https://github.com/hinthornw/trustcall/blob/main/trustcall/_base.py#L1535)
- [LangGraph `ToolNode` 文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents#toolnode)

## Observations

- [fact] `create_extractor()` 返回组合 `Runnable`，内部包含已编译的 LangGraph，而不是预构建的 `ToolNode`。
- [fact] 验证节点对模型的 tool call 执行 `model_validate`，并产生成功或错误的 `ToolMessage`。
- [fact] 更新路径默认让模型调用 `PatchDoc`，在本地将 JSON Patch 应用到旧对象，再做 schema 校验。
- [insight] `result["messages"]` 是过滤后的最终 AI 消息，不是模型、校验与修复的完整消息轨迹。

## Relations

- relates_to [[LangGraph State：TypedDict 与 Pydantic 的选择]]
- relates_to [[WorkerState、Send 与多 Schema 节点]]