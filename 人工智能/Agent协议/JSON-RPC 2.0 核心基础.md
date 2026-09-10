---
title: JSON-RPC 2.0 核心基础
type: guide
permalink: my-kb/人工智能/agent-协议/json-rpc-2.0-核心基础
tags:
- JSON-RPC
- MCP
- ACP
- 协议
created: 2026-09-07
source: 对话整理
---

# JSON-RPC 2.0 核心基础

## 定位

JSON-RPC 2.0 是一种使用 JSON 表示远程方法调用的轻量协议。它规定消息如何描述调用、结果和错误，但不定义具体方法的业务语义。

本地函数调用：

```text
result = add(2, 3)
```

对应的 JSON-RPC 调用：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "add",
  "params": {
    "a": 2,
    "b": 3
  }
}
```

## 三种核心消息

### Request

Request 表示需要响应的方法调用：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "method": "user/get",
  "params": {
    "userId": "u-123"
  }
}
```

- `jsonrpc`：协议版本，固定为 `"2.0"`
- `id`：请求标识，用来匹配响应
- `method`：需要调用的方法
- `params`：方法参数，可省略

多个请求可以并发处理，响应不一定按请求发送顺序返回，因此必须通过 `id` 匹配。

### Response

成功响应：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "result": {
    "name": "Alice"
  }
}
```

失败响应：

```json
{
  "jsonrpc": "2.0",
  "id": 42,
  "error": {
    "code": -32601,
    "message": "Method not found"
  }
}
```

响应中的 `id` 必须对应原 Request。一个响应不能同时包含 `result` 和 `error`。

常见标准错误码：

| 错误码 | 含义 |
|---:|---|
| `-32700` | JSON 解析错误 |
| `-32600` | 无效请求 |
| `-32601` | 方法不存在 |
| `-32602` | 参数无效 |
| `-32603` | 服务端内部错误 |

### Notification

Notification 是不需要响应的单向通知，因此没有 `id`：

```json
{
  "jsonrpc": "2.0",
  "method": "progress/update",
  "params": {
    "progress": 0.5
  }
}
```

适用于进度、日志、状态变化、取消操作和流式内容更新。

## Request 与 Notification 的判断

| 类型 | 是否有 `id` | 是否需要响应 |
|---|---|---|
| Request | 是 | 是 |
| Notification | 否 | 否 |

最关键的判断规则：**有 `id` 的调用需要 Response；没有 `id` 的调用是 Notification，接收方不能回复。**

## ACP 中的典型流程

```text
Client -> Agent: initialize          Request
Agent  -> Client: initialize 结果     Response
Client -> Agent: session/new         Request
Agent  -> Client: session/new 结果    Response
Client -> Agent: session/prompt      Request
Agent  -> Client: session/update     Notification
Agent  -> Client: session/update     Notification
Agent  -> Client: prompt 完成结果      Response
```

其中 `session/prompt` 可以保持未完成状态，Agent 在处理期间通过多个 `session/update` Notification 持续报告消息片段、进度和工具调用，最后才返回与原请求 `id` 对应的 Response。

`session/cancel` 也是 Notification：客户端要求取消任务，但不为该取消消息等待响应；原来的 `session/prompt` Request 最终会以取消原因结束。

## 协议分层

```text
传输层：stdio / HTTP / WebSocket
协议封装：JSON-RPC 2.0
具体协议：MCP / ACP
业务能力：工具、资源、会话、文件、终端、权限
```

JSON-RPC 只负责：

- 消息结构
- 方法调用
- 结果和错误表示
- 请求与响应的关联
- 单向通知

JSON-RPC 不负责：

- `session/prompt` 的具体语义
- MCP Tool、Resource、Prompt 的定义
- ACP Session、权限和 diff 的定义
- 选择 stdio、HTTP 或 WebSocket 等传输方式

## 实现的基本过程

```text
接收并解析 JSON
-> 识别 method
-> 校验 params
-> 执行方法
-> 对 Request 返回 result 或 error
-> 对 Notification 不返回消息
```

## 核心结论

1. Request 有 `id`，需要 Response。
2. Notification 没有 `id`，不需要也不能得到 Response。
3. Response 使用 `id` 与 Request 匹配，不能依赖消息顺序。
4. Response 包含 `result` 或 `error`，二者不能同时出现。
5. JSON-RPC 规定通信形式，MCP 和 ACP 负责定义业务语义。

## Observations

- [fact] JSON-RPC 2.0 使用 `id` 关联 Request 与 Response，允许多个请求并发执行且乱序返回。
- [fact] Notification 不包含 `id`，接收方不能为其发送 Response。
- [insight] MCP 与 ACP 都可以复用 JSON-RPC 的调用机制，但二者通过不同的方法名、参数和状态机定义各自的业务语义。
- [insight] 理解 JSON-RPC 时应把传输方式、RPC 消息封装和上层协议语义分开，避免将 stdio 或 HTTP 误认为 JSON-RPC 本身的一部分。

## Relations

- foundation_for [[Model Context Protocol（MCP）]]
- foundation_for [[Agent Client Protocol（ACP）]]