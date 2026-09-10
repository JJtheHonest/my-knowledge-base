---
title: Model Context Protocol（MCP）
type: guide
permalink: my-kb/人工智能/agent-协议/model-context-protocol-mcp
tags:
- MCP
- Agent协议
- 工具调用
- 上下文
created: 2026-09-10
source: 对话整理；MCP 2026-07-28 规范
---

# Model Context Protocol（MCP）

## 定位

MCP（Model Context Protocol）是 **AI 应用连接外部工具和数据源的标准协议**。它统一能力发现、调用和内容表示，但不规定模型如何推理、Host 如何管理对话或何时必须调用工具。

```text
AI 应用（Host）
    ↕ MCP
文件系统 / GitHub / 数据库 / 知识库 / 外部服务
```

MCP 通常使用 JSON-RPC 2.0 表示协议消息。

## 核心参与者

- **MCP Host**：用户实际使用的 AI 应用，负责对话、模型调用、上下文、权限、工具注册与路由。
- **MCP Client**：Host 内部维护一条 MCP Server 连接的组件。
- **MCP Server**：通过 MCP 暴露工具、资源或提示模板的程序，可以运行在本地或远程。

通常一个 Host 为每个 Server 创建一个独立 Client：

```text
MCP Host
├── Client A ↔ 文件系统 Server
├── Client B ↔ GitHub Server
└── Client C ↔ 数据库 Server
```

**LLM 通常不是 MCP Client，也不会直接发送 MCP 请求。** 模型表达工具调用意图，Host 将它转换为 MCP 请求并路由给相应 Client。

## Server 的三种核心能力

| 能力 | 含义 | 典型控制者 |
|---|---|---|
| **Tool** | 可执行的操作 | 模型选择，Host 裁决 |
| **Resource** | 可寻址、可读取的上下文数据 | 应用 |
| **Prompt** | 可发现、可参数化的消息模板 | 用户 |

可以记为：

```text
Prompt 告诉模型“如何做”
Resource 提供模型“需要知道什么”
Tool 让模型“能够采取行动”
```

### Tool

Tool 是具有名称、描述和输入 Schema 的远程函数接口，例如搜索、执行 SQL、创建 Issue。`inputSchema` 使用 JSON Schema 描述参数，使 Host 和 Server 可以校验模型生成的调用参数。

核心方法：

- `tools/list`：发现某个 Server 提供的工具定义。
- `tools/call`：调用该 Server 的一个具体工具。

Tool 不一定有副作用；查询也可以设计为 Tool。它与 Resource 的主要区别是：Tool 强调**执行操作**，Resource 强调**定位数据对象**。

### Resource

Resource 是使用 URI 标识的可读取数据，例如：

```text
file:///project/README.md
knowledge://notes/mcp
database://schema/users
```

- `resources/list` 返回资源描述，不一定包含实际内容。
- `resources/read` 根据 URI 返回资源内容。
- Resource Template（如 `database://users/{userId}`）描述一类参数化资源。

URI 用于唯一标识和寻址，不一定是真实网络地址。通常由 MCP Server 解释 URI。

### Prompt

Prompt 是 Server 发布的可复用消息模板，通过 `prompts/list` 发现、通过 `prompts/get` 填入参数并取得消息。它不执行任务，只为模型组织任务说明、示例或上下文引用。

## 工具发现不是广播

`tools/list` 始终是**一个 Client 对一个 Server**的普通请求。多个 Server 场景下，Host 可以并发调用各 Client，再在 Host 层汇总：

```text
Client A → Server A: tools/list → tools A
Client B → Server B: tools/list → tools B
Client C → Server C: tools/list → tools C

Host 汇总：tools A + tools B + tools C
```

因此，“向多个 Server 扇出请求并汇总”是 Host 的实现逻辑，不是 MCP 的广播机制。

工具名只要求在单个 Server 内唯一。多个 Server 出现同名工具时，Host 必须建立带来源的工具注册表或重命名，以便将后续 `tools/call` 路由到正确的 Client。

## 一次完整工具调用

```text
1. Client 向各自 Server 请求 tools/list
2. Host 汇总工具并建立“模型工具名 → Client → Server 原名”的注册表
3. Host 将 MCP Tool Definition 转换为模型供应商的工具格式
4. LLM 根据用户问题生成工具调用意图
5. Host 校验参数、权限和安全策略
6. Host 找到对应 Client，发送 MCP tools/call
7. Server 执行工具并返回 Tool Result
8. Host 将结果转换为模型 API 的工具结果消息
9. LLM 根据工具结果继续调用工具或生成最终回答
```

责任边界可浓缩为：

```text
LLM 提议
Host 裁决、转换和路由
Client 传递
Server 执行
LLM 解释结果
```

MCP Server 通常只看见工具名与参数，不必知道完整对话。Agent 的“模型生成 → 工具执行 → 结果回填”循环由 Host 管理，不属于 MCP 本身。

## Tool Result 与错误

Tool Result 可以包含文本等内容，也可以提供结构化 JSON。错误分为两层：

- **JSON-RPC 协议错误**：请求格式错误、未知方法或未知工具等，请求本身无法成立。
- **工具执行错误**：方法和工具均有效，但业务执行失败，例如日期无效或外部 API 超时。此类错误应提供可操作信息，便于模型调整参数后重试。

## Content Block

MCP 使用带类型的 Content Block 统一承载内容，常见类型包括：

- `text`：文本
- `image`：Base64 图片及 MIME 类型
- `audio`：Base64 音频及 MIME 类型
- `resource_link`：资源引用，仅提供 URI
- `resource`：嵌入资源，同时提供 URI 和实际内容

```text
Resource Link：给地址
Embedded Resource：给地址和内容
```

Resource 是可寻址的数据对象，Content Block 是消息载体。ACP 所说的“尽可能复用 MCP 类型”，主要就包括对这类 JSON 内容表示的复用，而不是复用 MCP 的完整业务流程。

## 能力发现与变化通知

Client 可通过 `server/discover` 了解 Server 支持的协议版本和 Tools、Resources、Prompts 等能力。当前规范的请求通过 `_meta` 携带协议版本和 Client 能力；旧版教程常见的 `initialize` 流程不可与新版消息直接混用。

当工具或资源列表变化时，Server 可以发送变更通知。通知通常只表示缓存失效，Client 仍需重新调用 `tools/list` 或 `resources/read` 获取完整最新状态。

## 传输层（简略）

- **stdio**：Host 启动本地 Server 子进程，通过 stdin/stdout 传递 MCP 消息。
- **Streamable HTTP**：Client 通过 HTTP 访问独立部署的远程 Server。

两者传递相同的 MCP/JSON-RPC 语义；传输层只解决消息如何送达。stdio Server 的 stdout 应专用于协议消息，日志写入 stderr。

## 权限与安全边界（简略）

MCP 只统一调用方式，不自动保证安全。本地 MCP Server 本质上是以用户权限运行的程序，远程 Server 则涉及账号授权与数据外发。基本原则是：只连接可信 Server、授予最小权限、高风险操作由 Host 审批，并将 Tool 描述、Resource 内容和 Server 返回值视为不可信输入。

**模型选择调用工具，不等于模型有权执行工具。**

## 实践参考

实现和接入 MCP Server 的实践参考：

- [MCP Integration](https://book.zyh.lol/06-mcp-integration/)

## 核心结论

1. MCP 标准化 AI Host 与外部能力之间的连接，不负责完整 Agent 推理流程。
2. Host 管理模型和多个 Client；Client 通常与 Server 一一对应；Server 提供具体能力。
3. Tool 是操作，Resource 是可寻址数据，Prompt 是消息模板。
4. `tools/list` 不是广播；Host 可以并发查询多个 Server 并在自身内部汇总。
5. 模型只提出调用意图，Host 负责格式转换、校验、授权和路由，Server 负责执行。
6. 工具结果必须回填模型上下文，模型才能继续推理或生成最终回答。
7. Content Block 是 MCP 的通用内容载体，也是理解 ACP 类型复用的关键。

## Observations

- [insight] MCP 的核心价值是把“模型供应商的工具调用格式”与“外部能力的实现”解耦，转换与编排集中在 Host。
- [insight] `tools/list` 的多 Server 汇总发生在 Host 层，每条 MCP 请求仍是 Client 与 Server 的一对一通信。
- [insight] Tool、Resource 和 Prompt 的边界应按交互语义判断，而不能简单按是否读取数据判断。
- [insight] MCP 规范能力通信，Agent 循环、权限决策和最终用户体验属于 Host 的职责。

## Relations

- builds_on [[JSON-RPC 2.0 核心基础]]
- foundation_for [[Agent Client Protocol（ACP）]]