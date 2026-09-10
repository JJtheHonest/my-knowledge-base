---
title: Agent Client Protocol（ACP）
type: guide
permalink: my-kb/人工智能/agent-协议/agent-client-protocol-acp
tags:
- ACP
- Agent协议
- 编码智能体
- OpenCode
created: 2026-09-10
source: ACP v1 官方文档；OpenCode 官方源码 dev 分支
---

# Agent Client Protocol（ACP）

## 定位

ACP（Agent Client Protocol）标准化**代码编辑器/交互客户端与编码 Agent 之间的通信**，作用类似于 LSP 对编辑器与语言服务器集成的标准化。

```text
编辑器 / IDE / 终端 UI
          ↕ ACP
       编码 Agent
```

它使一个兼容 ACP 的 Agent 能被不同编辑器复用，也使编辑器不必为每个 Agent 单独开发专有集成。ACP 基于 JSON-RPC 2.0，并尽可能复用 MCP 的 JSON 内容表示。

本文以稳定的 ACP v1 为主；ACP v2 目前属于草案。

## 核心参与者

- **ACP Client**：编辑器、IDE 或其他交互界面，负责用户输入、过程展示、权限交互以及可选的文件系统和终端能力。
- **ACP Agent**：调用 LLM、管理上下文、规划任务、使用工具并完成代码工作的智能体。

Client 和 Agent 是**逻辑角色**，不一定对应两个独立产品或进程。一个完整编码 Agent 产品可以同时包含自带 UI 和 Agent 核心；只有跨产品通过 ACP 连接时，协议边界才会明确显现。

## 与 MCP 的区别和组合

```text
用户
 ↕
ACP Client（编辑器）
 ↕ ACP
ACP Agent（编码智能体）
 ↕ MCP
外部工具与数据源
```

| MCP | ACP |
|---|---|
| Host/Agent 连接外部能力 | 编辑器连接完整编码 Agent |
| 核心对象是 Tool、Resource、Prompt | 核心对象是 Session 和 Prompt Turn |
| `tools/call` 请求 Server 执行工具 | Tool Call Update 主要向 UI 报告执行过程 |
| 不管理完整 Agent 对话 | 管理会话、消息流和任务生命周期 |
| 不定义编码 UI | 支持计划、权限、diff、终端等编码体验 |

**MCP 让 Agent 获得外部能力，ACP 让编辑器控制并呈现 Agent 的完整工作过程。**

ACP Client 可以在创建会话时把 `mcpServers` 配置交给 Agent，由 Agent 连接这些 MCP Server。

## 双向调用

ACP 并不是 Client 单向调用 Agent。双方都可以发起 JSON-RPC Request 或 Notification。

Client 调用 Agent的典型方法：

- `initialize`：协商协议版本和能力
- `session/new`：创建会话
- `session/load` / `session/resume`：恢复会话
- `session/prompt`：发送用户任务
- `session/cancel`：取消当前 Prompt Turn

Agent 调用 Client 的典型方法：

- `session/request_permission`：请求用户授权工具操作
- `fs/read_text_file` / `fs/write_text_file`：使用 Client 的文件系统能力
- `terminal/create`、`terminal/output`、`terminal/kill`：使用 Client 的终端能力

编辑器掌握用户界面和本地环境，因此 Agent 需要通过反向调用来请求这些能力。

## Session

Session 表示一条独立对话及其任务上下文。`session/new` 通常携带：

- `cwd`：会话的主要工作目录，必须是绝对路径
- `mcpServers`：希望 Agent 在本会话中连接的 MCP Server
- 可选的额外工作区根目录

Agent 返回唯一的 `sessionId`。后续 Prompt、取消、权限和更新都通过它归属到相应会话。

## Prompt Turn

一次 Prompt Turn 从 Client 发送 `session/prompt` 开始，到 Agent 返回最终 `stopReason` 结束。任务执行期间，Agent 通过多个 `session/update` Notification 流式报告状态：

```text
Client → Agent: session/prompt Request

Agent → Client: session/update（计划）
Agent → Client: session/update（消息片段）
Agent → Client: session/update（工具调用）
Agent → Client: session/update（工具状态 / diff / 终端）
...

Agent → Client: 原 session/prompt 的最终 Response
```

核心结构是：

```text
一个长期 Request
+ 多个流式 Notification
+ 一个最终 Response
```

`session/prompt` 的输入是 `ContentBlock[]`，可以包含文本、图片、资源链接或嵌入资源。这里复用了 MCP 的内容表示思想，但方法和会话流程由 ACP 定义。

常见 `stopReason` 包括 `end_turn`、`max_tokens`、`refusal` 和 `cancelled`。

## Tool Call：执行与展示的区别

LLM 请求工具后，Agent 负责实际执行，并通过 `session/update` 向 Client 报告：

```text
pending → in_progress → completed / failed
```

更新可以包含：

- 工具调用标题与 `read`、`edit`、`search`、`execute` 等类型
- 原始输入和输出
- 涉及的文件及行号
- 普通 Content Block
- 文件 diff
- 可在 Client 中显示实时输出的终端 ID

ACP 不要求工具一定来自 MCP。它可以是 Agent 内置工具、MCP Tool，或 ACP Client 提供的文件和终端能力。ACP Tool Call 的重点是**统一向 Client 呈现执行过程**；MCP `tools/call` 的重点是**请求 MCP Server 执行工具**。

## 权限请求

敏感操作前，Agent 可以调用 `session/request_permission`，由 Client 展示选项并返回用户选择。常见选项类型：

```text
allow_once
allow_always
reject_once
reject_always
```

职责边界是：Agent 提出权限请求，Client 根据用户设置和选择决定结果。Client 也可以按策略自动允许或拒绝。

## 一轮编码任务

```text
1. Client 与 Agent 初始化并交换能力
2. Client 创建 Session，指定 cwd 和可选 MCP Servers
3. Client 发送 session/prompt
4. Agent 调用 LLM 并形成计划
5. Agent 通过 session/update 流式报告消息和工具状态
6. Agent 按需请求权限，或使用 Client 的文件/终端能力
7. Agent 执行工具，并报告 diff、终端输出和结果
8. 工具结果回到 LLM，继续下一轮推理
9. Agent 返回最终 stopReason
```

## 协议角色与产品内部架构

产品中的 UI 与 Agent 核心即使承担 ACP Client/Agent 的相似职责，也不代表它们内部实际使用 ACP。协议角色描述的是责任边界，协议实现描述的是线上消息格式，两者必须区分。

### OpenCode 的情况

根据 OpenCode 官方 `dev` 分支源码，**OpenCode 自带 TUI 与核心之间不使用 ACP**，而是使用 OpenCode 特定的、开源但非行业标准的 Server API 和内部 Worker RPC：

```text
OpenCode TUI
  ↓ OpenCode SDK / HTTP 风格 API
内部 Worker RPC
  ↓
OpenCode Server 与 Agent 核心
```

默认本地模式使用虚拟地址 `http://opencode.internal`，将请求通过 Worker 转交给 `Server.Default().app.fetch`；事件由 `GlobalBus` 经内部 RPC 转给 TUI。

`opencode acp` 是独立的对外适配入口：

```text
外部 ACP Client
  ↓ ACP over stdio / NDJSON
OpenCode ACP Adapter
  ↓ @opencode-ai/sdk
OpenCode Server API
  ↓
OpenCode Agent 核心
```

因此准确表述是：

```text
OpenCode TUI ↔ 核心：OpenCode 内部协议
外部编辑器 ↔ OpenCode Agent：ACP
```

OpenCode 源码参考：

- [TUI 命令入口](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/tui.ts)
- [TUI Worker](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/tui/worker.ts)
- [内部 RPC](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/util/rpc.ts)
- [ACP 命令入口](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/cmd/acp.ts)
- [ACP Agent 实现](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/acp/agent.ts)

## 核心结论

1. ACP 标准化编辑器/交互客户端与编码 Agent 的通信。
2. Client 管理用户体验和本地能力，Agent 管理模型、上下文、规划和工具执行。
3. ACP 是双向协议，Agent 可以反向请求 Client 的权限、文件和终端能力。
4. Prompt Turn 是一个长期 Request、多个流式更新和一个最终 Response。
5. ACP Tool Call Update 主要报告执行过程，不等同于 MCP 的工具执行请求。
6. ACP 和 MCP 互补：前者连接编辑器与 Agent，后者连接 Agent 与外部能力。
7. 协议角色是逻辑职责；产品内部可以使用自有协议，只在对外集成时提供 ACP 适配层。

## Observations

- [insight] ACP 的核心价值不是创造 Session、权限或进度展示，而是把编码 Agent 产品已有的交互职责标准化，使 Agent 核心可以脱离自带 UI 被其他客户端复用。
- [insight] `session/update` 将 Agent 的内部工作过程转换成客户端可渲染的结构化事件，是 ACP 编码体验互操作性的核心。
- [insight] ACP 与 MCP 对“工具调用”的关注层次不同：ACP 关注 Agent 工作流的可见性，MCP 关注外部能力的发现与执行。
- [fact] OpenCode 的 TUI 使用 OpenCode 内部 Server API 和 Worker RPC；`opencode acp` 则是将 ACP 消息适配到 OpenCode SDK/Server API 的独立入口。

## Relations

- builds_on [[JSON-RPC 2.0 核心基础]]
- builds_on [[Model Context Protocol（MCP）]]