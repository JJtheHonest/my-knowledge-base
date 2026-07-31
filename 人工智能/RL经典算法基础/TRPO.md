---
title: TRPO
type: note
permalink: my-kb/人工智能/rl-经典算法基础/trpo
tags:
- 强化学习
- 策略优化
- TRPO
- 信任域
- KL散度
---

# TRPO

**TRPO（Trust Region Policy Optimization）2015**

**核心：运用残差思想，定义出优势函数A**

$A^{\pi}(s,a)=Q^{\pi}(s,a)-V^{\pi}(s)$

需要注意的是，Q和V的区别和关系，Q是某个动作的价值，而V是所有动作按概率加权的平均价值：

$V^{\pi}(s)=\pi(a_1|s)Q^{\pi}(s,a_1) + \pi(a_2|s)Q^{\pi}(s,a_2) + \dots + \pi(a_n|s)Q^{\pi}(s,a_n)$

因此，$A^\pi$表示的就是某个动作高于平均动作的优势程度。

定义新旧策略的期望回报之间的差距：

$J(\theta_{\text{new}})=J\left(\theta_{\text {old}}\right)+\mathbb{E}_{s \sim d^{\pi_{\text{new}}}}\left[\sum_{a}\pi_{\text{new}}(a|s)A^{\pi_{\text {old}}}(s, a)\right]$

这里A表示的就是：根据新策略选择的动作a，高于旧策略的平均动作的程度。

在旧策略的期望回报J_old上叠加这一差值，就可以得到新策略的期望回报J_new。

现在问题在于：d^new很难获得，严格来说，他是新策略下的状态访问概率，即：

$d^{\pi_\text{new}}(s) = (1-\gamma) \sum_{t=0}^{\infty} \gamma^t P(s_t = s | \pi_\text{new})$

如果直接求解该分布，需要用新策略来收集样本$\tau$，这是很难实现的。因此，基于**“每次策略更新的变化都很小，因此两个策略之间的状态访问分布变化可忽略”**的假设，我们直接使用旧策略d^old代替新策略d^new，定义一个“替代”优化目标（Surrogate Objective）$L_\theta(\theta_{\text{new}})$：

$J(\theta_{\text{new}}) \approx J\left(\theta_{\text {old}}\right)+\mathbb{E}_{s \sim d^{\pi}}\left[\sum_{a}\pi_{\text{new}}(a|s)A^{\pi_{\text {old}}}(s, a)\right]$

$L_\theta(\theta_{\text{new}}) = \mathbb{E}_{s \sim d^{\pi}, a\sim \pi}\left[\frac{\pi_{\text{new}}(a|s)}{\pi_{\text{old}}(a|s)}A^{\pi_{\text {old}}}(s, a)\right]$

现在我们可以对“替代”优化目标求解了，但是注意：这一切都基于“策略相似”假设。因此，TRPO加入了一个信任约束，使策略每次更新都使用小步幅。这也是算法名中“trust region”的由来。这个约束简单来说，就是使得新旧策略之间的平均KL散度始终小于阈值$\delta$：

$\max_{\theta_{\text{new}}} \ L_\theta(\theta_{\text{new}}) \quad \text{subject to} \quad \mathbb{E}_{s \sim d^\pi} \left[ D_{KL} \left( \pi_\text{old}(\cdot|s) \| \pi_\text{new}(\cdot|s) \right) \right] \leq \delta$

经过上述的设计，我们现在通过只选择“满足L>0或L≥0”的新策略，理论上可以保证策略性能单调增或单调不减。(macOS里打出≥是option + .)

除此之外，TRPO还使用了泰勒展开近似、共轭梯度、线性搜索等方法来更快求解。对优势函数A的估计运用了多步时序差分的思想，使用GAE方法（广义优势估计，Generalized Advantage Estimation），这些数学技巧和算法核心思想关系不大，本文只记个人思路，详细可以参看远林师兄的笔记[trpo](https://kmddxkwxdr.feishu.cn/wiki/NqTTwFZfMinSArk3HkMc1SVVnOg)

## Relations

- builds_on [[Policy Gradient 推导]]
- uses [[GAE]]
- improved_by [[PPO]]