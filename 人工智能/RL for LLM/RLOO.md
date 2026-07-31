---
title: RLOO
type: note
permalink: my-kb/人工智能/rl-for-llm/rloo
tags:
- 强化学习
- RLHF
- RLOO
- REINFORCE
- 大模型
---

# RLOO

这篇论文的核心结论可以概括为一句话：**在 LLM 的 RLHF 阶段，PPO 并不是必需甚至可能不是最合适的工具；更简单的 REINFORCE / RLOO 风格方法，在更低计算复杂度下可以取得更好或至少更稳的效果。**

## 论文反对的“默认共识”

传统 RLHF 流程通常把 **PPO** 当作强化学习阶段的标准算法，但作者认为这是从传统 Deep RL 场景中“照搬”来的默认选择。PPO 需要同时维护 policy、reference model、reward model、critic/value model，计算成本高，也对超参数较敏感。论文明确指出，PPO 在 RLHF 中的许多设计动机并不成立。([ar5iv](https://ar5iv.org/html/2402.14740v2))

## 作者的核心判断：LLM RLHF 更像 bandit，不像传统 MDP

PPO 通常把“每生成一个 token”看作一个 action，把“部分生成序列”看作 state。但在 RLHF 中，奖励模型通常只对**完整回答**打分，中间 token 并没有真实的人类偏好奖励。因此作者认为，把整个 completion 当作一个单一 action，更符合实际，也更简单。也就是说，RLHF 中的偏好优化可以被简化为一个以 prompt 为初始状态、完整生成结果为动作的 bandit 问题。([ar5iv](https://ar5iv.org/html/2402.14740v2))

## 为什么简单 REINFORCE 反而有效？

传统上 REINFORCE 被认为方差高，不如 actor-critic / PPO 稳定。但作者认为，在 LLM 对齐中，模型不是从随机策略开始训练，而是从预训练模型和 SFT 模型 warm start。策略已经很强，概率质量集中在较少的合理 token 上，因此传统 RL 中“必须强力降方差”的问题没有那么严重。相反，PPO 的 GAE、critic/value bootstrap 等机制虽然降低方差，却可能引入不必要的 bias。论文实验显示，在 RLHF 环境里，减少 bias 往往比进一步降方差更重要。(ar5iv)

## 论文主推的方法：RLOO

作者重点使用 **REINFORCE Leave-One-Out, RLOO**。它的想法是：对同一个 prompt 采样多个回答，每个回答的 reward 减去其他回答 reward 的平均值作为 baseline，再对这些样本的梯度估计取平均。这样既保留了 REINFORCE 的简单性和无 critic 特点，又通过多样本 baseline 降低方差。([ar5iv](https://ar5iv.org/html/2402.14740v2))

和 PPO 相比，RLOO 的优势是：

- 不需要 value / critic model；

- 不需要 token-level advantage estimation；

- 不依赖 PPO clipping 机制；

- 直接优化完整序列级 reward；

- 能更充分利用多个 online samples。

## 最重要的启示

这篇论文不是说“RLHF 不需要 RL”，而是说：**RLHF 不一定需要复杂的 PPO；在线 RL 仍然有价值，但应该使用更贴合 LLM 偏好学习特性的简单算法。**

它的实践含义是：做 LLM 对齐时，可以优先考虑 **sequence-level REINFORCE / RLOO** 这类方法，而不是默认使用 PPO。尤其在资源有限、调参能力有限、希望降低训练复杂度的场景下，RLOO 可能是一个很有吸引力的替代方案。论文最终的核心主张是：**回到基础的 policy gradient，并针对 LLM 的 warm-start、sequence-level reward、KL 约束等特点做适配，可能比沿用复杂 PPO 更有效。**

## Relations

- builds_on [[Policy Gradient 推导]]
- alternative_to [[PPO]]
- relates_to [[GRPO]]
- discussed_in [[用于 LLM 的 RL 实践技术细分]]