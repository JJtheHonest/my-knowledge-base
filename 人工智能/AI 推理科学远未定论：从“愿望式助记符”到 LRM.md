---
title: AI 推理科学远未定论：从“愿望式助记符”到 LRM
type: note
permalink: my-kb/人工智能/ai-推理科学远未定论从-愿望式助记符-到-lrm
tags:
- AI推理
- 思维链
- LRM
- Lean
- AlphaProof
- AlphaEvolve
- wishful-mnemonics
created: 2026-08-15
source: Pavlus, "Is AI Reasoning Right for the Wrong Reasons?" (2026) 及延伸讨论
---

# AI 推理科学远未定论：从"愿望式助记符"到 LRM

## 背景与主旨

这篇博客质疑一个直觉：AI 能否"推理"如今显得前所未有地真实，但直觉可能出错，**这门科学远未尘埃落定**。核心张力在于：LRM（Large Reasoning Model）在可验证任务（数学、代码）上成绩斐然，但"推理轨迹"（思维链）是否真的对应着模型内部在做逻辑推理，学界仍有严重分歧。

## 关键引文与解读

### 1. "参差的智能"（jagged intelligence）
"Whatever — I guess that's just 'jagged intelligence' for you" —— AI 行话，指"能用的时候就能用"：能力分布参差不齐，好用与失灵并存，缺乏稳定一致的智能轮廓。

### 2. 虚假的理论比没有理论更糟
"A fake theory is worse than admitting that we don't have a theory." —— 与其用站不住脚的解释自我麻醉，不如诚实承认我们还没有解释。

### 3. "这不是科学，这是投资"
有人为这些模型强行宣称能力辩护的理由是"先宣称，反正以后可能成真"。批评者反驳：**那不是科学，那是投资** —— 用愿景代替证据。

### 4. 过度敬畏思维链有代价
即便思维链结果可验证，把它看得太神圣也会阻碍科学家寻找更好的方法来引导 LRM 输出准确结果——"我们可能错失了一些尚未探索的机会"。

### 5. Kambhampati 的"兔子洞"论
认真对待 AI 推理轨迹的"含义"，被比作相信**地心说或以太**：可能是一条科学的死胡同，把精力浪费在对轨迹字面意义的解读上。

## 历史回响：愿望式助记符（Wishful Mnemonics）

本博客引用了 McDermott 1976 年论文《Artificial Intelligence Meets Natural Stupidity》中的经典段落，构成整个批评传统的历史源头：

- **问题**：程序员用"UNDERSTAND""GOAL"这类高深词汇给主循环、数据结构起名，属于**乞题**（begging the question）——预先假设了要证明的结论。
- **危害**：名字会误导人（尤其作者本人），制造虚假的成就感，掩盖程序并未真正实现智能的事实。
- **建议**：改叫无意义的中性编号"G0034"，然后问自己："G0034 真的实现了'理解'的一部分吗？"
- **一句话总结**：别用玄乎的名字给程序贴金，先看看代码实际干了什么。

**与当下的呼应**：给模型推理轨迹贴"推理"之名，与当年给主循环贴"UNDERSTAND"之名，是同一类危险——用愿望式命名掩盖了验证的缺失。

## 延伸讨论：Lean 与 AI 证明工具链

### Lean 是什么（简要）
Lean 是交互式定理证明器/依赖类型语言：**命题=类型，证明=构造对象，编译器=机器裁判**。模型提交候选证明，Lean 逐行校验并给出精确报错，绝不靠"感觉对"。由此为 LRM 提供**自动、严格、可无限扩展的监督信号**——"生成—验证—反馈迭代"。

### AlphaProof Nexus 与 Lean
AlphaProof Nexus 正依赖 Lean 做形式化验证。这是目前"可验证推理"路线最成功的实践。

### AlphaEvolve 与 Lean 的关系（澄清）
- **AlphaEvolve 本身不用 Lean**：它是进化式编码智能体，在"程序空间"里由 LLM 变异、用确定性 fitness 函数评分。
- **Lean 在其下游**：论文中的完整流水线是 AlphaEvolve 找构造 → Deep Think 推导证明 → **AlphaProof 用 Lean 形式化验证**。
- 把"AlphaEvolve 找候选 + 自动生成 Lean 证明"做成全自动，仍是未完成的目标。

### AlphaEvolve 与 AlphaFold 的异同
- **共同"基因"**：都是 DeepMind 式"明确评分 + 迭代优化"的设计哲学。
- **实质差异**：AlphaFold 是**预测**（监督学习，序列→结构）；AlphaEvolve 是**搜索/优化**（进化算法，在程序空间演化出新构造，无需标注数据）。其思想更接近 FunSearch/AlphaZero，而非 AlphaFold。

## Observations

- [insight] "参差的智能"（jagged intelligence）刻画了 AI 能力分布不均：能用时就用得成，失灵时毫无征兆，不能据此断言稳定的智能 #jagged-intelligence
- [insight] "虚假的理论比没有理论更糟"——对推理机制的解释若不严谨，宁可承认未知 #认识论
- [insight] "先宣称能力，以后反正会成真"是投资心态而非科学心态，本质是用愿景替代证据 #科学方法
- [insight] 过度敬畏思维链可能阻碍更优的引导 LRM 的方法被发现 #思维链
- [insight] Kambhampati 认为认真对待推理轨迹的字面含义是"兔子洞"，类比相信地心说/以太 #推理轨迹
- [fact] 愿望式助记符（wishful mnemonics）源自 McDermott 1976《Artificial Intelligence Meets Natural Stupidity》，指用"UNDERSTAND""GOAL"等名词给程序命名从而乞题 #wishful-mnemonics
- [insight] 给推理轨迹贴"推理"之名 ≈ 给主循环贴"UNDERSTAND"之名，愿望式命名掩盖验证缺失，历史在重演 #wishful-mnemonics
- [fact] AlphaEvolve 本身不用 Lean；Lean 由 AlphaProof 使用，用于把 Deep Think 推导的证明做形式化验证 #AlphaEvolve #Lean #AlphaProof
- [fact] AlphaEvolve 是进化式编码智能体（LLM 变异 + 确定性 fitness 函数），与 AlphaFold 的监督预测范式实质不同，更接近 FunSearch/AlphaZero #AlphaEvolve #AlphaFold

## Relations

- relates_to [[GRPO]]
- relates_to [[用于 LLM 的 RL 实践技术细分]]
- relates_to [[PPO]]