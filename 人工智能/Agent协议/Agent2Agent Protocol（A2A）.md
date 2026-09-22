---
title: Agent2Agent Protocol（A2A）
type: guide
permalink: my-kb/人工智能/agent-协议/agent2-agent-protocol-a2-a
tags:
- A2A
- Agent协议
- 多智能体
- 任务协作
created: 2026-09-11
source: A2A 1.0.0 官方规范与核心概念文档
---

# Agent2Agent Protocol（A2A）

## 定位

A2A（Agent2Agent Protocol）是独立 AI Agent 之间进行能力发现、任务委派、消息交换和状态跟踪的开放协议。当前正式版本为 **1.0.0**。

A2A 面向可能由不同公司、框架、语言和模型构建的 Agent，并遵循**黑盒执行（Opaque Execution）**原则：远程 Agent 声明能力、接口、状态与结果，但不必暴露内部模型、记忆、System Prompt、工具或推理实现。

```text
对外：能力、任务状态、消息和成果
对内：模型、记忆、工具和规划实现
```

## 交互角色

- **A2A Client（Client Agent）**：代表用户或其他系统发起请求的一方，可以是应用、服务或另一个 Agent。
- **A2A Server（Remote Agent）**：暴露 A2A 接口并处理任务的远程 Agent 或 Agent 系统。

Client/Server 是针对**一条交互关系**定义的动态角色，不是全局身份或能力等级：

```text
Agent A --A2A--> Agent B --A2A--> Agent C
Client A          Server B
                  Client B        Server C
```

同一 Agent 可以同时作为上游请求的 Server 和下游委派的 Client。并非所有 Agent 都必须实现两种角色：成为 Server 需要发布 Agent Card 并暴露接口；成为 Client 需要具备发现和调用其他 Agent 的能力。

## 主 Agent 与编排边界

系统中常见一个直接面向用户的主 Agent（Orchestrator）：

```text
用户
 ↓
主 Agent
 ├── A2A → 航班 Agent
 ├── A2A → 酒店 Agent
 └── A2A → 财务 Agent
 ↓
汇总结果并回复用户
```

但**主 Agent 和多 Agent 编排不是 A2A 协议规定**。任务拆分、Agent 选择、依赖调度、并行策略、失败恢复、冲突处理和结果汇总都属于 Agent、工作流引擎或多 Agent 框架的实现责任。A2A 只提供描述 Agent、委派任务、跟踪状态及交换成果的标准接口。

## Agent Card

Agent Card 是远程 Agent 发布的 JSON 能力名片，常见发现地址为：

```text
/.well-known/agent-card.json
```

它描述：

- Agent 身份、描述和版本
- 服务地址、协议绑定及协议版本
- Skills
- 支持的输入输出 MIME 类型
- Streaming、Push Notification 等能力
- 认证与安全要求

Agent Card 支持“读取和理解某个 Agent 的能力”，但 A2A 不强制规定如何在全网找到 Agent Card 的 URL；地址可以来自配置、Registry、目录服务或人工输入。

### Skill 与 MCP Tool

A2A Skill 是高层业务能力，例如“安排商务旅行”或“审核保险理赔”；MCP Tool 通常是具体操作，例如 `search_flights` 或 `query_policy`。一个 Skill 可以在远程 Agent 内部经过多轮推理并调用多个 Tool 才完成。

## 核心数据对象

### Message 与 Part

Message 是 Client 与 Remote Agent 的一次交流，包含 `messageId`、`role` 和一个或多个 Part。

`role` 表示当前 A2A 交互的消息方向：

- `user`：请求方/Client 侧发送的消息
- `agent`：提供服务的 Remote Agent 侧发送的消息

`user` 不保证发送者是真实人类。主 Agent 代表用户调用专业 Agent 时，它发送的 Message 仍使用 `role=user`。该字段不是身份认证信息。

Part 是最小内容单元，可以承载：

- `text`：文本
- `data`：结构化 JSON
- `url`：外部文件引用
- `raw`：内联二进制数据

并可携带 MIME 类型、文件名和元数据。

### Task

Task 是 A2A 的核心，是具有唯一 `id` 和生命周期的**有状态工作单元**。复杂、长时间或需要多轮交互的请求通常返回 Task；简单且无需跟踪的请求可以直接返回 Message。

Task 通常包含：

- `id`：任务标识
- `contextId`：相关交互的逻辑分组
- `status`：当前状态
- `history`：相关消息历史
- `artifacts`：任务成果
- `metadata`：扩展数据

### Artifact

Artifact 是 Task 产生的正式交付物，例如报告、行程单、图片、代码补丁或结构化订单，由一个或多个 Part 组成。

```text
Message：Agent 在说什么
Artifact：Agent 正式交付了什么
```

## Task 生命周期

典型流程：

```text
SUBMITTED → WORKING → COMPLETED
```

状态可分为：

- **活跃**：`SUBMITTED`、`WORKING`
- **可继续的中断**：`INPUT_REQUIRED`、`AUTH_REQUIRED`
- **终态**：`COMPLETED`、`FAILED`、`CANCELED`、`REJECTED`

`REJECTED` 表示 Agent 不接受或拒绝继续任务；`FAILED` 表示已接受但执行失败。

### 多轮补充

Task 处于 `INPUT_REQUIRED` 或 `AUTH_REQUIRED` 时，Client 可以向**同一个 taskId**发送后续 Message，使其继续：

```text
WORKING
  ↓
INPUT_REQUIRED
  ↓ Client 补充 Message
WORKING
  ↓
COMPLETED
```

### Task 不可变性

Task 一旦进入终态就不能恢复。针对已完成结果的修改或追问，应在同一个 `contextId` 下创建新 Task，并通过 `referenceTaskIds` 或 Artifact 引用表达依赖。

这样每个 Task 都是清晰、可审计的输入输出单元，也允许后续任务并行执行。

## Context

`contextId` 将多个相关 Task 和独立 Message 归入同一长期目标：

```text
Context：规划芬兰旅行
├── Task 1：预订航班
├── Task 2：根据航班预订酒店
└── Task 3：根据航班安排活动
```

同一个 Remote Agent 可以用 `contextId` 管理内部对话状态或 LLM 上下文，但它只是**关联标识，不是协议定义的共享内存**。A2A 不保证：

- 新 Task 自动得到全部旧历史和 Artifact
- 并行 Task 自动看到彼此的中间结果
- 不同 Remote Agent 理解或共享同一个 `contextId`
- 存在所有 Task 都能读写的标准 Context 数据对象

依赖重要时，Client 应显式传递 `referenceTaskIds`、Artifact 引用和必要的结构化上下文。跨 Agent 的全局状态通常由主 Agent或编排系统维护。

```text
contextId：告诉同一 Remote Agent“这些交互有关联”
referenceTaskIds：指出“具体依赖哪些任务”
Artifact 引用：指出“具体使用哪个成果”
编排器状态：维护跨 Agent 的全局事实与依赖
```

## 一次基本协作

```text
1. Client 获取 Remote Agent 的 Agent Card
2. 检查 Skills、输入输出类型、接口和认证要求
3. Client 发送 Message
4. 简单请求由 Agent 直接返回 Message
5. 复杂请求由 Agent 创建并返回 Task
6. Task 从 SUBMITTED 进入 WORKING
7. Agent 内部独立推理并使用自己的工具
8. 缺少信息或授权时进入 INPUT_REQUIRED / AUTH_REQUIRED
9. Client 向同一 taskId 补充 Message
10. Agent 产生一个或多个 Artifact
11. Task 进入 COMPLETED 或其他终态
```

## 更新与异步方式

Task 是权威状态，A2A 提供三种获取更新的方式：

- **Polling**：Client 定期调用 `GetTask`，实现简单但实时性较低。
- **Streaming**：通过流式连接持续接收 Task 状态和 Artifact 更新，适合用户在线等待。
- **Push Notification**：Remote Agent 回调 Client 提供的 Webhook，适合耗时很长或 Client 离线的任务。

三种方式可以组合使用。例如流式连接断开后用 `GetTask` 恢复状态，并为超长任务配置 Push Notification。

Client 可发送 `CancelTask`，但这只是取消请求；必须检查返回的 Task State，不能假设请求送达就等于已经取消。

## 与 MCP、ACP 的关系

```text
用户
 ↓
编辑器 / 应用
 ↕ ACP
主 Agent
 ├── MCP → 工具、数据库、API
 └── A2A → Remote Agent
              └── MCP → 该 Agent 自己的工具
```

| 协议 | 核心问题 |
|---|---|
| **ACP** | 用户界面如何连接和呈现编码 Agent？ |
| **MCP** | Agent 如何发现和调用工具、资源？ |
| **A2A** | 一个独立 Agent 如何委派任务并与另一个 Agent 协作？ |

```text
ACP：使用 Agent
MCP：使用工具
A2A：委托 Agent
```

A2A Agent 与 Tool 的关键差异在于：Agent 能自主推理、规划、维护任务状态并进行多轮协商，而 Tool 通常执行边界明确的具体操作。

## 与 OpenCode Subagent 的关系

OpenCode 的 Primary Agent 通过 Task 工具调用 Subagent，与 A2A 都体现了任务委派模式，也都可以创建独立子任务并返回结果。

但 OpenCode Subagent 是同一产品内部的 Agent 编排和 child session 机制；它不通过 Agent Card 发现，也不要求暴露 A2A 接口。A2A 则用于可能跨框架、平台、团队或公司的独立 Agent 系统之间互操作。因此两者**概念相似，但实现上不是同一机制**。

## 协议边界

A2A 规定：

- 如何描述 Agent 及其 Skills
- 如何发送 Message
- 如何创建、继续、查询和取消 Task
- 如何交换状态、Part 和 Artifact
- 如何进行流式或异步更新

A2A 不规定：

- 如何拆分用户目标
- 如何选择最合适的 Agent
- 任务依赖如何调度
- 如何合并或仲裁多个 Agent 的结果
- 如何维护跨 Agent 的全局记忆

## 核心结论

1. A2A 面向独立 Agent 之间的高层任务委派与协作。
2. Agent Card 声明身份、Skills、接口、内容类型和安全要求。
3. Client 与 Server 是每条交互关系中的动态角色，同一 Agent 可以同时承担两者。
4. Message 表示交流，Task 表示可跟踪工作，Artifact 表示正式成果。
5. `role=user` 表示请求方消息，不证明发送方是人类。
6. `INPUT_REQUIRED` 和 `AUTH_REQUIRED` 支持同一 Task 内的多轮补充。
7. 终态 Task 不可恢复；修改和追问应在同一 Context 中创建新 Task。
8. `contextId` 提供逻辑连续性，但不是自动共享内存。
9. Agent 发现元数据属于 A2A；多 Agent 编排策略不属于 A2A。
10. MCP 面向具体工具，A2A 面向能够自主完成复杂任务的 Agent。

## Observations

- [insight] A2A 的核心抽象不是远程函数，而是具有状态、可中断、可恢复交互并能产生 Artifact 的 Task。
- [insight] A2A Client/Server 是关系角色而非固定身份，这允许 Agent 在委派链中同时服务上游并调用下游。
- [insight] `contextId` 提供同一 Remote Agent 内的语义连续性；跨 Agent 的共享状态与依赖仍必须由编排器显式维护。
- [insight] Agent Card 解决“已知地址后如何理解 Agent”，但全局搜索、评分和选择 Agent 仍属于 Registry 或编排系统。

## Relations

- builds_on [[JSON-RPC 2.0 核心基础]]
- complements [[Model Context Protocol（MCP）]]
- complements [[Agent Client Protocol（ACP）]]