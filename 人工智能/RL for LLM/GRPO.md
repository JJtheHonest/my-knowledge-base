---
title: GRPO
type: note
permalink: my-kb/人工智能/rl-for-llm/grpo
tags:
- 强化学习
- RLHF
- GRPO
- 大模型
- 优势估计
---

# GRPO

# 算法核心思想（一句话总结）

GRPO 的核心是：**对每个题目（prompt）同时采样一组输出（group），用该组内输出的平均/相对得分作为基线来估计优势，从而省去对独立 value/critic 网络的依赖，并在 token 级别以类似 PPO 的方式优化策略，同时用对照（reference）策略的 KL 作为正则项**。这种“组内相对（group-relative）”的优势估计非常契合奖励模型常常以“同题目下输出对比”为训练信号的事实。

---

# 算法要解决的问题（提出问题）

在把 RL（尤其是 PPO）用于大型语言模型（LLM）微调时，有几个实际问题：

1. **资源开销**：PPO 通常需要同时训练或保存一个 value 网络（critic）以估计基线 ($V_\psi$)，这对于大模型在显存/算力上代价高。

2. **奖励性质**：许多 reward model（尤其是基于比较/偏好的 reward）天然以“比较同一题目下不同输出”的相对分数为训练方式，直接用单样本的 scalar reward 去做 token 级 advantage 估计并不完全契合这一性质。

3. **基线估计的稳定性**：当 reward 很稀疏或在 token 级别有较大噪声时，单独训练的价值网络在有限数据上可能不稳定或过拟合。

---

# 分析问题与 GRPO 的解决思路（问题→分析→解决）

## 问题分析（Why PPO struggles）

PPO 的策略梯度在实现上依赖优势函数 ($A_t$)，而 ($A_t$) 通常通过 GAE（Generalized Advantage Estimation）基于一个学习到的 value 函数 ($V_\psi$) 来计算：

$A_t = \sum_{l=0}^{\infty} (\gamma\lambda)^l \delta_{t+l},\quad \delta_t = r_t + \gamma V_\psi(s_{t+1}) - V_\psi(s_t),$ 

在 LLM 的 token-level 场景中，状态、回报和依赖关系非常高维，训练一个稳定的 value 函数既吃显存又不易收敛；另外 reward model 常以“同 prompt 的输出比较”来产出分数，直接把这些分数当做标量去训练价值函数并不总是最自然的选择。

## GRPO 的关键想法（解决方案）

GRPO 的关键步骤与思想如下：

1. **同题目采样一组输出**：对每个问题 (q)（或 prompt），从旧策略 ($\pi_{\theta_\text{old}}$) 采样一个大小为 (G) 的输出组 (${o_1,\dots,o_G}$)。

2. **按组计算分数并用组均值作基线**：对组内每个输出运行 reward model 得到标量 ($r_i$)，以组平均或组内相对方式得到基线（即把组均值作为 baseline，用个体的得分与组均值的差作为“相对优势”）。因此不再需要单独训练 value 网络。

    - 论文称之为 **group relative advantage estimation**，记为 ($\hat{A}_{i,t}$)（token 级别的“组相对优势”）。

3. **与 PPO 相似的 clip / token 级 objective**：在策略更新上仍使用与 PPO 类似的 token-level 目标（包含 clipping 操作），但把 ($A_t$) 换成 ($\hat{A}{i,t}$*)，*并把 KL 正则直接加入目标（或在目标中以参数 ($\beta$) 对参考策略 ($\pi_{\text{ref}}$) 做 KL 惩罚），从而保证策略不偏离参考策略太远。

4. **迭代流程 + reward model 的 replay**：作者构建了一个迭代流程（Algorithm 1）：采样→用 reward model 评分→计算组相对优势→在若干 GRPO 步中更新策略→持续以 replay 机制更新 reward model（用历史数据的一部分），以减少 reward 模型退化风险。

---

# 关键数学公式（从论文摘录并解释）

论文给出的 GRPO 目标（在简化 ($\pi_{\theta_\text{old}}=\pi_\theta$) 的情形下的写法）可以写成（按论文符号重写为 LaTeX 形式，便于理解）：

策略目标（形式化）——论文式（排版上略作清晰化）：
$J_{\mathrm{GRPO}}(\theta)=\mathbb{E}_{[q\sim P(Q), \{o_i\}_{i=1}^G\sim \pi_{\theta_\text{old}}(\cdot|q)]}\left[\frac{1}{G}\sum_{i=1}^G\frac{1}{|o_i|}\sum_{t=1}^{|o_i|}\operatorname{clip_{term}}\bigg(\frac{\pi_\theta(o_{i,t}\mid q,o_{i,<t})}{\pi_{\theta_\text{old}}(o_{i,t}\mid q,o_{i,<t})},\hat{A}{i,t}\bigg) - \beta\mathrm{D_{KL}}(\pi_\theta| \pi_{\mathrm{ref}})\right]$
 其中 ($\operatorname{clip_{term}}(\cdot)$) 表示 PPO 的 clip 操作（min( ratio * advantage, clipped_ratio * advantage)），($\hat{A}_{i,t}$) 是**基于组的相对优势估计**（group-relative advantage），论文并在附录中给出其具体计算流程。梯度系数（paper 中称为 gradient coefficient）可写成（paper 的等价表达）：
$\text{GCGRPO}(q,o_i,t) = \hat{A}{i,t} + \beta\left(\frac{\pi_{\text{ref}}(o_{i,t}\mid \cdot)}{\pi_\theta(o_{i,t}\mid \cdot)} - 1\right).$
 论文（附录 A.1.6）给出了更为精确的 token-级梯度形式与系数定义；主要思想如上：把基线替换为组内信息，并把 KL 正则通过系数加入梯度计算。

**关于 (**$\hat{A}_{i,t}$**)（组相对优势）的直观解释**：常见做法是把某输出 ($o_i$) 的总 reward ($r_i$) 与组内平均 ($\bar r = \frac{1}{G}\sum_j r_j$) 做差，再把该差值分摊或扩展到 token 级别（或结合 token-level 得分）以得到 token-level 的 ($\hat{A}_{i,t}$)。论文指出，这与 reward model 多以“同题目下输出比较”训练的性质高度匹配，因此这样估计更自然且统计效率更高。

---

# 关键创新点（为什么新颖、有价值）

1. **去掉 value/critic，转为组基线（节省资源）**：直接用同一 prompt 下多输出的组均值作为 baseline，避免了额外训练 value 网络，从而**显著降低显存与训练开销**，这对于在有限资源上微调大模型尤为重要。论文及后续实现说明明确把“减少对 value 模型的依赖”作为主要动机之一。

2. **与 reward model 的训练/性质更契合**：因为 reward model（尤其用于生成任务的比较型 reward）往往是在“同一 prompt 下的输出对比”数据上训练的，GRPO 的组内相对优势直接利用了这一结构，使得优势估计与奖励来源一致，理论与实践上都更稳健。

3. **在 token 级别保留 PPO 风格的 clipping 与 KL 正则**：保留了 PPO 的稳定化技巧（clip 操作、KL 正则化），同时把优势换成“组相对优势”，兼顾稳健性与资源效率。论文在实验中显示，这种组合能在数学推理任务上带来显著提升（DeepSeekMath-RL 相比仅 SFT/Instruction-tuning 有提升）。

4. **迭代训练与 reward replay 机制**：为避免 reward model 随着策略漂移而退化，论文采用 replay（把部分历史数据混入训练）并交替更新 reward 与策略，从工程上增强了稳定性。

---

# 与其他算法的关系（基于 / 借鉴）

- **基于 PPO**：GRPO 是在 PPO 框架下的变体 — 它保持了 PPO 的核心（ratio、clip、token-level 优化和 KL-regularization 思路），但替换了价值估计方式。可以把 GRPO 看做 **“PPO − critic + group-baseline + group-advantage”**。

- **与 DPO / RFT / Online RFT / DPO 的关系**：论文在 Section 4 与附录中把 GRPO 与拒绝采样微调（RFT）、直接偏好优化（DPO）等方法放在同一分析框架下比较，指出 GRPO 属于“在线采样 + model-based reward”并使用 model objective 的一类方法（而 DPO 用 pairwise loss，RFT 用拒绝采样过滤，PPO/GRPO 则用模型梯度优化）。因此 GRPO 更接近 **在线 PPO 家族**，但对 baseline & advantage 的处理上做了本领域针对性修改。

---

# 实践建议 / 复现实验时的要点与陷阱

1. **组大小 (G) 的选择**：(G) 太小则组内均值方差大、优势估计噪声高；(G) 太大则采样/评分开销变大（reward model 多次评估同一 prompt）。论文里实验使用了一定的 (G)（参见 Algorithm 1），复现实验时建议先尝试中等 (G)（例如 4–16）并做 ablation。

2. **reward model 的稳定性**：GRPO 依赖于 reward model 在同一组输出之间给出相对一致性评分。务必使用 replay 机制或混入历史数据来训练 reward model，防止 reward model 随策略快速漂移导致信号崩坏。论文在 Algorithm 1 中采用了 10% 历史数据的 replay 方案作为一个例子。

3. **KL 正则系数 (\\beta)**：论文直接在目标中引入与 reference policy 的 KL 项作为正则（而不是仅把 KL 加入 reward），这在实践中是控制策略变动、避免崩坏的有效手段。要通过小范围 sweep 找到合适的 ($\beta$)。

4. **token-level vs sequence-level reward**：论文把优势分配到 token 级别（token-level objective），这需要把 sequence reward 转换为 token-level signal（例如把总体 reward 均摊或用序列中每 token 的累积评分）。在实现细节上需小心（论文在附录给出了处理方式）。

5. **与 PPO 的稳定性对比实验**：论文报告 GRPO 在数学任务上的提升与训练成本下降（DeepSeekMath-RL 相比 Instruct 模型有明显提高）；在复现时应把训练步骤、batch 大小、采样数、reward 模型更新频率等与 PPO 做严格对齐以公平比较。

---

# 优点 / 局限性的总结（简短）

**优点**

- 显著降低对 value/critic 的训练需求、节省显存与计算。

- 优势估计与比较式 reward model 天然匹配，统计效率更高。

- 保留 PPO 的稳定化（clip、KL）优点，同时工程实现上更轻便。

**局限 / 需注意**

- 需要对同一 prompt 重复采样多次（增加 reward model 评估开销）——工程权衡需仔细设计组大小 (G)。

- 如果 reward model 本身不稳定或不以比较式训练为主，组内基线的优势可能不如训练 value 网络来的稳定。

---

# 参考、出处（我读到并核对的关键原文与实现说明）

- DeepSeekMath 论文（arXiv / PDF）：**DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models**（作者 DeepSeek-AI）——GRPO 的定义、Algorithm 1、附录 A.1.6 的推导我均直接核对自论文 PDF。

- DeepSeek 的 GitHub / 代码仓库（项目资源、收敛曲线、复现脚本与超参说明）。([GitHub](https://github.com/deepseek-ai/DeepSeek-Math?utm_source=chatgpt.com))

- 社区 / 文档对 GRPO 的实现说明（例如 HuggingFace 的 GRPO Trainer 文档）用于工程实践参考。([Hugging Face](https://huggingface.co/docs/trl/main/en/grpo_trainer?utm_source=chatgpt.com))

- 论文在后续工作（如 DeepSeek-R1 的训练说明 / Nature 报道）中也提到 GRPO 被用于 R1 系列微调流程的事实（可作为 GRPO 在生产级别应用的佐证）。([Nature](https://www.nature.com/articles/s41586-025-09422-z?utm_source=chatgpt.com))

## Relations

- builds_on [[PPO]]
- contrasts_with [[GAE]]
- relates_to [[RLOO]]
- discussed_in [[用于 LLM 的 RL 实践技术细分]]