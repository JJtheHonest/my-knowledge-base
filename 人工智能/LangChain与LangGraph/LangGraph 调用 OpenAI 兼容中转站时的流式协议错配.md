---
title: LangGraph 调用 OpenAI 兼容中转站时的流式协议错配
type: guide
permalink: my-kb/人工智能/lang-chain-与-lang-graph/lang-graph-调用-open-ai-兼容中转站时的流式协议错配
tags:
- LangGraph
- LangChain
- OpenAI兼容协议
- SSE
- 故障排查
created: 2026-09-22
source: 本地调试记录
---

# LangGraph 调用 OpenAI 兼容中转站时的流式协议错配

## 问题现象

在 LangGraph 中通过 `init_chat_model()` 调用一个 OpenAI 兼容中转站，并绑定 `add`、`multiply`、`divide` 等工具。模型第一次执行 `invoke()` 时偶发以下异常：

```text
AttributeError: 'str' object has no attribute 'model_dump'
During task with name 'llm_call'
```

异常发生在 `llm_call` 内部，模型响应尚未成功转换成 `AIMessage`，因此此时还没有进入 LangGraph 的 `tool_node`。同一份代码有时又可以正常执行，表现出明显的不稳定性。

## 调用链

```text
LangGraph StateGraph
  -> LangChain llm_with_tools.invoke()
  -> langchain-deepseek
  -> langchain-openai
  -> OpenAI Python SDK
  -> 中转站 /v1/chat/completions
```

`langchain-deepseek` 底层复用了 OpenAI 兼容客户端。配置 `base_url` 后，请求实际发送给中转站，而不是 OpenAI 官方接口。

## 根因

中转站没有稳定遵守 OpenAI Chat Completions 协议。

对 `/v1/chat/completions` 的实测结果表明，即使请求明确包含：

```json
{
  "stream": false
}
```

服务端仍返回：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
```

正文也是 SSE 流：

```text
data: {"object":"chat.completion.chunk", ...}

data: [DONE]
```

这构成了协议错配：客户端请求非流式 JSON，服务端却返回流式 SSE。HTTP 状态码为 `200` 只表示服务端认为请求成功，不代表响应格式符合约定。

## OpenAI 协议中的两种响应模式

### 非流式模式

当 `stream` 缺省或为 `false` 时，服务端应一次性返回：

```http
Content-Type: application/json
```

响应主体应为完整的 `chat.completion` JSON 对象，其中包含 `choices`、`message`、`tool_calls` 和 `finish_reason` 等字段。OpenAI SDK会将其解析为 `ChatCompletion` Pydantic 模型。

### 流式模式

当 `stream=true` 时，服务端应返回：

```http
Content-Type: text/event-stream
```

响应由多个 `chat.completion.chunk` 事件组成。文本内容和工具调用参数可能被拆分到多个 `delta` 中，客户端需要持续读取并合并这些片段，直到收到 `data: [DONE]`。

例如，一个工具调用可能被拆成：

```text
片段 1：function.name = "add"
片段 2：arguments = "{\"a\":"
片段 3：arguments = "3,\"b\":4}"
```

LangChain在流式模式下会将这些片段合并成完整的 `AIMessage.tool_calls`。

## 为什么报错出现在 model_dump

`langchain-openai` 的非流式处理逻辑期望得到两种对象之一：

```python
response: dict | openai.BaseModel
```

其处理逻辑相当于：

```python
response_dict = (
    response
    if isinstance(response, dict)
    else response.model_dump(...)
)
```

由于中转站返回了与请求模式不匹配的 SSE，OpenAI SDK没有获得预期的 `ChatCompletion` 模型，上层最终拿到的是 `str`。LangChain随后对字符串调用 `.model_dump()`，于是产生：

```text
AttributeError: 'str' object has no attribute 'model_dump'
```

因此，`.model_dump()` 只是暴露问题的位置，根因是中转站响应协议和客户端请求模式不一致。

## 为什么有时可以成功

同一代码偶尔成功，通常意味着中转站行为不一致，可能包括：

- 中转站后端有多个网关节点，节点实现或版本不一致；
- 中转站根据负载动态切换不同上游供应商；
- 部分路径正确处理 `stream=false`，部分路径强制使用 SSE；
- 代理层在转发时偶发遗漏或覆盖 `stream` 参数；
- 服务更新期间不同节点尚未同步。

若成功请求返回 `application/json`，失败请求返回 `text/event-stream`，即可进一步确认问题来自中转站协议行为不稳定，而不是 LangGraph 图随机出错。

## 临时解决方案

根据中转站实际返回 SSE 的行为，在模型初始化时启用流式模式：

```python
model = init_chat_model(
    model="deepseek-v4.1-flash-0910",
    api_key=os.environ["SHUAI_API_KEY"],
    base_url=os.environ["OPENAI_BASE_URL"],
    streaming=True,
)
```

启用后，LangChain会按 SSE 读取 `ChatCompletionChunk`，合并文本和工具调用增量，再生成完整 `AIMessage`。实测可以完成以下循环：

```text
HumanMessage
  -> 模型生成 add(a=3, b=4) 工具调用
  -> tool_node 返回 7
  -> 模型生成最终答案
```

该设置是适配中转站当前行为的 workaround，不是对服务端协议错误的根本修复。

## 根本解决方案

服务端应严格区分两种模式：

```text
stream=false
  -> Content-Type: application/json
  -> 完整 chat.completion

stream=true
  -> Content-Type: text/event-stream
  -> 多个 chat.completion.chunk + [DONE]
```

若中转站无法保证该行为，应联系服务商修复或更换接口。

## 排查方法

遇到类似问题时，优先检查 HTTP 层，而不是先修改 LangGraph 图：

1. 确认请求中的 `stream` 值。
2. 检查响应状态码和 `Content-Type`。
3. 检查正文是完整 JSON，还是以 `data:` 开头的 SSE。
4. 对比成功请求和失败请求的响应头。
5. 记录请求 ID、服务节点或模型路由信息。
6. 分别测试普通对话、流式输出、工具调用和结构化输出。

关键判断：

```text
stream=false + application/json       -> 正常
stream=true  + text/event-stream      -> 正常
stream=false + text/event-stream      -> 服务端协议错配
```

## 开发实践建议

学习和调试 LangChain、LangGraph 时，优先使用模型厂商官方 API：

- 官方示例和 SDK默认以官方协议行为为基准；
- 更容易区分应用代码问题与代理兼容问题；
- 工具调用、结构化输出和流式增量字段通常更完整；
- 错误状态码、错误正文和请求 ID更规范；
- SDK升级后的兼容性通常更可靠。

中转站声称“OpenAI 兼容”并不代表完整实现了所有协议细节。普通文本对话可用，也不能证明工具调用、流式输出和结构化输出完全兼容。

同时应遵守密钥管理规范：API Key 从环境变量读取，不写入源码；如果密钥曾经出现在源码、日志或对话中，应立即轮换。

## Observations

- [fact] 实测中转站在显式 `stream=false` 时仍返回 `Content-Type: text/event-stream` 和 `chat.completion.chunk` SSE 数据。
- [fact] LangChain异常发生在模型响应解析阶段，早于 LangGraph 的工具节点执行。
- [insight] HTTP `200` 不等于 OpenAI 兼容协议实现正确，还必须核对响应 Content-Type、对象类型和字段结构。
- [insight] 间歇性成功更可能表示中转站节点或上游路由行为不一致，而不是确定性的 LangGraph 图逻辑错误。
- [decision] 当前可使用 `streaming=True` 适配实际响应；学习和基准调试阶段优先使用官方 API。
- [decision] API Key 使用环境变量管理，任何曾经明文暴露的密钥都应轮换。

## Relations

- relates_to [[LangGraph]]
- relates_to [[LangChain]]
- relates_to [[OpenAI API]]