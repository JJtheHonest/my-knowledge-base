---
title: Agent 发现与多 Agent 编排
type: guide
permalink: my-kb/人工智能/agent-协议/agent-发现与多-agent-编排
tags:
- A2A
- Agent发现
- 多Agent
- 编排
created: 2026-09-11
source: A2A 编排设计整理
---

# Agent 发现与多 Agent 编排

## 协议边界

```text
A2A 提供：Agent Card、Message、Task、Artifact 和任务状态接口
编排系统负责：搜索、评估、选择、拆解、调度、容错与结果合并
```

A2A 将 Agent 暴露为可发现、可调用、可跟踪的黑盒任务服务，但不规定完整的多 Agent 编排系统。

## Agent 发现管线

Agent 发现不是单一步骤，而是“获取候选、过滤兼容性、评估适用性、最终选择”的管线。

### 1. 获取候选 Agent

| 方式 | 适用场景 |
|---|---|
| 静态配置 | 内部系统、Agent 少且关系固定 |
| Well-Known URI | 已知服务域名 |
| Registry | Agent 较多，需要按 Skill、标签或组织搜索 |

Well-Known 地址通常为：

```text
https://example.com/.well-known/agent-card.json
```

A2A 定义 Agent Card，但不规定 Registry 的标准查询 API。

### 2. 过滤兼容性

编排器读取 Agent Card 后，应先按确定性条件过滤：

- 是否包含所需 Skill
- 输入、输出 MIME 类型是否兼容
- 是否支持所需的 Streaming 或 Push
- 协议版本和绑定是否兼容
- 当前身份是否满足认证要求
- 是否属于允许调用的组织或网络

### 3. 评估适用性

多个 Agent 都声称具备所需能力时，可按以下因素排序：

```text
score =
  能力匹配度
+ 历史成功率
+ 结果质量
- 预计成本
- 延迟
- 当前负载
- 安全与合规风险
```

Agent Card 是 Agent 的自我声明，不是能力证明。生产系统还应结合健康检查、历史指标、人工认证、契约测试和调用反馈。

### 4. 最终选择

```text
规则过滤 → 候选排名 → 选择首选与备用 Agent
```

LLM 适合理解用户意图和 Skill 描述；可信范围、数据边界和预算则应由确定性策略控制，不能允许 LLM 不受限制地调用任意公共 Agent。

## 控制面与执行面

```text
控制面：Registry、Agent Card 缓存、身份认证、策略、评分、健康状态
执行面：SendMessage、Task、状态更新、Artifact、取消和重试
```

Agent Card 更新较少，适合按 HTTP 缓存语义缓存；Task 状态属于运行时数据，需要持续跟踪。

## 常见编排模式

### 中心化 Orchestrator

```text
用户 → 主 Agent → 多个专业 Agent → 主 Agent → 用户
```

主 Agent 维护全局目标、任务依赖和最终结果。这种结构最容易控制、审计和调试，通常应优先采用。

### 分层委派

```text
主 Agent
  → 旅行 Agent
      → 航班 Agent
      → 酒店 Agent
```

适合大型领域，但链路越深，成本、延迟、权限传播和故障定位越困难，因此需要限制最大深度和总任务数。

### Handoff

当前 Agent 将主要交互转交给另一个 Agent，常用于客服转接。系统必须明确最终回复的责任方、上下文移交方式，以及原 Agent 是否继续跟踪。

### 共享事件或黑板

多个 Agent 围绕共享工作区读取任务并发布成果。共享状态、并发控制和冲突解决由编排框架负责，并非 A2A `contextId` 自动提供。

## 从用户目标生成任务图

编排器通常将目标拆成有向无环图（DAG）：

```text
Task A：确定航班
   ├── Task B：根据到达时间选择酒店
   └── Task C：根据到达时间安排接送
Task B + Task C
   → Task D：生成完整行程
```

每个节点应明确：

- 负责 Agent
- 输入契约与预期 Artifact
- 依赖节点
- 超时与预算
- 重试策略
- 失败是否阻断全局任务

没有明确输出契约时，各 Agent 可能只返回无法可靠验证和合并的自然语言。

## 全局状态与上下文传递

编排器应维护独立的全局运行状态：

```text
Global Run
├── 用户目标与约束
├── Agent 选择记录
├── Task 依赖图
├── 各 Remote Agent 的 contextId/taskId
├── Artifact 索引
└── 预算、权限与审计信息
```

不同 Remote Agent 的 `contextId` 通常彼此独立。相同字符串不能实现跨 Agent 共享记忆；编排器应显式传递：

- 当前子任务所需的最小事实
- 上游 Artifact 或结构化摘要
- `referenceTaskIds`，仅限同一 Remote Agent
- 数据来源和版本
- 不可违反的用户约束

应遵循**最小充分上下文**原则：过少会丢失约束，过多会增加成本、数据泄露和提示注入风险。

## 调度循环

中心化编排器的核心流程为：

```text
发现并选择 Agent
→ 构建任务 DAG
→ 提交依赖已满足的节点
→ 通过 Streaming、Push 或轮询跟踪 Task
→ 保存状态与 Artifact
→ 解锁后继节点
→ 处理 INPUT_REQUIRED / AUTH_REQUIRED
→ 重试、切换备用 Agent 或降级
→ 验证并合并最终成果
```

不能仅凭 Remote Agent 的自然语言“完成了”判断成功，还必须检查 Task State、Artifact 是否存在，以及 Artifact 是否满足输出契约。

## 容错策略

需要区分三类失败：

| 类型 | 含义 | 典型策略 |
|---|---|---|
| 通信失败 | 请求是否送达未知 | 查询 Task、使用幂等标识，避免重复副作用 |
| Task 失败 | Remote Agent 明确执行失败 | 重试、切换备用 Agent 或终止依赖节点 |
| 结果不合格 | Task 已完成，但 Artifact 不符合契约 | 在同一 Context 创建修订 Task |

终态 Task 不能重启。若操作已经产生副作用，应执行补偿操作，例如取消预订，而不是直接重试。

## 实用架构

```text
用户
 ↓
主 Agent / Orchestrator
 ├── Registry Client：搜索 Agent Card
 ├── Policy Engine：控制信任、权限与预算
 ├── Planner：生成 Task DAG
 ├── Scheduler：处理并行、依赖、超时与重试
 ├── State Store：保存 contextId、taskId 与 Artifact
 ├── Validator：验证结构化结果
 └── A2A Clients：调用各 Remote Agent
```

其中，最后的 Agent 通信和部分发现元数据属于 A2A；其余组件均属于建立在 A2A 之上的编排系统。

## 核心原则

> 用 A2A 把 Agent 当作黑盒任务服务，用编排器维护全局事实、依赖、策略和最终责任。

## Observations

- [insight] Agent 发现应由确定性兼容性过滤与动态质量排序共同完成，Agent Card 本身不能充当能力证明。
- [decision] 多 Agent 系统优先采用中心化 Orchestrator，并为分层委派设置最大深度和任务总量限制。
- [insight] 编排器必须维护独立于各 Remote Agent Context 的全局状态，并显式传递最小充分上下文。
- [decision] 任务成功需要同时满足终态正确、Artifact 存在和输出契约验证通过。
- [insight] 通信失败、Task 失败和结果不合格具有不同语义，必须采用不同恢复策略。

## Relations

- builds_on [[Agent2Agent Protocol（A2A）]]
- complements [[Model Context Protocol（MCP）]]