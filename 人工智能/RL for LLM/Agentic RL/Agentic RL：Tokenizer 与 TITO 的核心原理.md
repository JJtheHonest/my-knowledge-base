---
title: Agentic RL：Tokenizer 与 TITO 的核心原理
type: note
permalink: my-kb/notes/agentic-rltokenizer-与-tito-的核心原理
tags:
- Agentic-RL
- TITO
- Tokenizer
- Harness
created: 2026-10-19
source: 与 Codex 的对话
---

# Agentic RL：Tokenizer 与 TITO 的核心原理

## 核心结论

**TITO（Token-In, Token-Out）的核心是保留模型实际接收、实际生成的 token 序列，使 rollout 与训练使用的 token、上下文和 logprob 对齐。** 它不禁止调用 tokenizer，而是避免用“解码后重新编码”的结果替代原始生成 token。

## Tokenizer 如何连接环境与模型

环境提供观测并执行动作；LLM 消费和生成 token。完整链路是：

```text
环境观测 → 消息格式化 / chat template → Tokenizer → 输入 token
模型生成 → 输出 token → Detokenizer / 动作解析 → 工具执行 → 新观测
```

- **Tokenizer / Detokenizer**：文本与 token ID 的转换。
- **Harness**：组织上下文、调用模型、解析动作、执行工具、推进交互循环。分词是其衔接的一部分，也可以由推理服务负责。
- **训练数据层**：保存每轮实际输入、输出 token，以及需要的 logprob 和 loss mask。

架构图里的 `(De)Tokenizer` 有时泛指适配层，但严格的分词组件不等于整个 harness。

## 为什么不能重新编码模型输出

假设词表中存在以下映射（ID 仅作示意）：

```text
101 → "he"；102 → "llo"；103 → "hello"

模型实际生成：[101, 102]
解码文本：    "hello"
重新编码：    [103]
```

解码丢失了原始分词边界。同一文本可以对应不同的 token 路径：原来是两次条件采样，替换后变成一次采样，对应的概率和训练动作都改变了。

**token ID 记录实际选了什么；logprob 记录策略给这个选择分配了多大概率。** 仅有 logprob 不足以恢复采样结果，即使保存全词表概率分布也一样。

## TITO 的典型实现

在历史只追加消息的情况下：

1. 首轮通过 chat template 编码初始上下文，发送 `input_ids`。
2. 从推理引擎直接获取生成 token ID 和对应的采样策略 logprob。
3. 保存“本轮输入 + 原始输出”作为 token checkpoint；另行解析输出供工具执行。
4. 工具返回后，只编码新增消息及必要模板边界，接到已有 token 前缀，再发给模型。

```text
初始输入 P → 模型输出 A → 新环境片段 E → 模型输出 B

第二轮输入：P + A + E
完整轨迹：  P + A + E + B
```

新增片段需要处理 role 标记、generation prompt 和模型特定边界，不能一概等同于 `encode(tool_result)`。Miles 的 session/checkpoint 与增量 tokenization 是这种实现思路的一个例子；具体源码细节随版本变化。

## 训练时必须守住的两个约束

**只对策略生成部分计算相应的 policy loss。** 例如：

```text
tokens：    [10,11, 20,21, 30,31,32, 40,41]
来源：      初始输入 模型A  工具结果等  模型B
loss_mask： [ 0, 0,  1, 1,  0, 0, 0,  1, 1]
```

这里是逐 token 的概念性 mask；实际张量布局取决于框架。采样策略 logprob 要与对应生成 token 逐个对齐。

**上下文可以压缩，但不能改写已发生的 rollout。** 如果第一轮是 `P → A`，压缩后的第二轮是 `S → B`，训练必须保留这两组真实条件关系，不能伪造成 `S → A → B`。可为压缩后的输入建立新 checkpoint，并按轮或分段保存记录；单条持续追加的 token 序列不再足以表达整个过程。

## Observations

- [insight] 文本相同不代表 token 路径相同；训练需要忠实于实际采样路径及其条件上下文。
- [insight] Append-only token checkpoint 是 TITO 的常见实现方式，核心要求是 rollout 与训练的 token 对齐。
- [decision] 对新环境内容正常编码；复用模型已生成的原始 token；保留每轮真实上下文与生成记录。

## Relations

- depends_on [[Policy Gradient 推导]]
- relates_to [[用于 LLM 的 RL 实践技术细分]]
