---
title: GAE
type: note
permalink: my-kb/人工智能/rl-经典算法基础/gae
tags:
- 强化学习
- 优势估计
- GAE
- 方差缩减
- 时序差分
---

# GAE

# Monte Carlo advantage

接续 [Policy Gradient 推导](https://kmddxkwxdr.feishu.cn/wiki/MxrgwvJqNiAeA1k9uzgcnn1Znie)，典型的 REINFORCE 梯度形式可以写作 $\nabla_\theta J(\theta) = \mathbb{E}_{\tau \sim p_\theta(\tau)} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_\theta(a_t|s_t) G_t \right]$

而 REINFORCE 的蒙特卡洛估计形式为：

$\widehat{\nabla_\theta J(\theta)} = \frac{1}{N} \sum_{i=1}^{N} \sum_{t=0}^{T_i-1} \nabla_\theta \log \pi_\theta(a_t^{(i)}|s_t^{(i)}) G_t^{(i)}$

它的优点是简单、直接，并且在理论上可以得到无偏估计；但缺点也很明显：方差通常很大。因为 $G_t$ 是从 t 时刻开始直到 episode 结束的完整回报，它会受到很多后续随机事件的影响，尤其在长轨迹任务中噪声非常大。

为了降低方差，我们可以在 $G_t$ 中减去一个只依赖状态 $s_t$ 的 [baseline](https://kmddxkwxdr.feishu.cn/wiki/MxrgwvJqNiAeA1k9uzgcnn1Znie#share-XrTddWguIohjsXxM862cFbCinzc) $b(s_t)$：

$\nabla_\theta J(\theta) = \mathbb{E} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_\theta(a_t|s_t) \left( G_t - b(s_t) \right) \right]$

一个自然的选择是令 baseline 等于状态价值函数：$b(s_t) = V^\pi(s_t)$

于是 $G_t - V^\pi(s_t)$ 就可以被理解为当前动作带来的相对优势，也就是我们定义的** 优势（advantage）**：

$A^\pi(s_t,a_t) = Q^\pi(s_t,a_t) - V^\pi(s_t)$

在采样中，$Q^\pi(s_t,a_t)$ 通常可以用 $G_t$ 来估计，因此：

$\hat{A}_t^{MC} = G_t - V^\pi(s_t)$

此时策略梯度变成：

$\nabla_\theta J(\theta) = \mathbb{E} \left[ \sum_{t=0}^{T-1} \nabla_\theta \log \pi_\theta(a_t|s_t) A_t \right]$

这个形式非常重要。它说明策略更新的核心不是单纯奖励高就增加概率，而是：

> 如果某个动作比当前状态下的平均水平更好，就提高它的概率；如果它比平均水平更差，就降低它的概率。
> 
> 

# TD advantage

虽然使用 $\hat{A}_t = G_t - V^\pi(s_t)$ 可以降低一定方差，但 $G_t$ 仍是完整的 Monte Carlo return：

$G_t = \sum_{k=0}^{T-t-1} \gamma^k r_{t+k}$，因此仍然可能具有较高方差。

为了进一步降低方差，可以使用一步 TD 误差来估计 advantage：

$\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)$

这个量也可以被看作一种 advantage 估计。

如果 $r_t + \gamma V(s_{t+1})$ 大于当前状态价值 $V(s_t)$，说明实际结果比价值函数原本预期的更好，那么这个动作就是有优势的；反之则说明这个动作低于预期。

因此，**一步 TD advantage** 可以写成：$\hat{A}_t^{(1)} = \delta_t$

但是一步 TD 估计虽然方差低，却依赖于价值函数 $V$ 的准确性。如果 $V$ 估计得不好，就会引入明显的偏差。

---

综上所述：

- Monte Carlo advantage：$\hat{A}_t^{MC} = G_t - V(s_t)$，偏差低，但方差高

- TD advantage：$\hat{A}_t^{(1)} = r_t + \gamma V(s_{t+1}) - V(s_t)$，方差低，但偏差高

于是问题变成：

> ### 能不能构造一种 advantage 估计方法，在 Monte Carlo 和 TD 之间做平滑折中？
> 
> 

# 多步 TD advantage

先定义 k-step return：

$G_t^{(k)} = r_t + \gamma r_{t+1} + \dots + \gamma^{k-1} r_{t+k-1} + \gamma^k V(s_{t+k})$

对应的 k-step advantage 为：

$\hat{A}_t^{(k)} = G_t^{(k)} - V(s_t)$

展开可以发现：

$\hat{A}_t^{(1)} = r_t + \gamma V(s_{t+1}) - V(s_t) = \delta_t$

$\hat{A}_t^{(2)} = \delta_t + \gamma \delta_{t+1}$

$\hat{A}_t^{(3)} = \delta_t + \gamma \delta_{t+1} + \gamma^2 \delta_{t+2}$

一般地：

$\hat{A}_t^{(k)} = \sum_{l=0}^{k-1} \gamma^l \delta_{t+l}$

因此，不同步数的 advantage 估计本质上就是不同长度的 TD residual 累积。

直观地讲：k-step advantage 可以被看成：

$\text{真实奖励累积 } k \text{ 步} + \text{第 } k \text{ 步之后用 } V \text{ 估计} - V(s_t)$

所以：

- k = 1：几乎完全依赖价值函数，方差低、偏差可能高； 

- k 增大：更多依赖真实采样奖励，方差变高、偏差变低； 

- k 到 episode 结束：不再 bootstrap，变成 Monte Carlo advantage。 

> ### 那么，看似 k-step TD advantage 已经实现在 Monte Carlo 和 TD 之间的平滑折中了，它还有什么缺点吗？
> 
> 

对于 k-step TD advantage：它只使用前 k 步的 TD residual，然后在第 k 步突然截断，用价值函数来 bootstrap，所以他的结构是：

$\text{前 } k \text{ 步用真实奖励} + \text{第 } k \text{ 步之后全部交给 } V$

这个截断点是硬性的。

比如 $k=5$，那么第 5 步以内的奖励会进入估计，第 6 步之后的影响就全部通过 $V(s_{t+5})$ 间接表示。

但实际任务里，并没有理由认为“第 5 步之前很重要，第 6 步之后突然不重要”。

所以固定 k-step 的问题是：

> 它确实折中了 TD 和 Monte Carlo，但这个折中依赖一个人为指定的、离散的、硬截断的 k。
> 
> 

# GAE

GAE，全称 Generalized Advantage Estimation，它的核心想法是：不要只选固定的 k-step advantage，而是把不同步数的 advantage 估计做**指数加权平均**：

写法(1)：$\hat A_t^{GAE} = (1-\lambda) \left( \hat A_t^{(1)} + \lambda \hat A_t^{(2)} + \lambda^2 \hat A_t^{(3)} + \cdots \right)$，这里的 $\lambda \in [0,1]$ 控制 bias-variance tradeoff。

把其中的 k-step advantage 展开，就得到了 GAE 另一种更常见的写法：

写法(2)：$\hat{A}_t^{GAE(\gamma,\lambda)} = \sum_{l=0}^{\infty} (\gamma \lambda)^l \delta_{t+l}$

其中 $\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)$

由写法(1)可以很直观地看到：

- 当 $\lambda=0$ 时，GAE 就是一步 TD advantage $\Rightarrow \hat A_t^{GAE} = \delta_t$

- 当 $\lambda \to 1$ 时，GAE 无限接近  $k\to\infty$ 的 k-step advantage，即无限接近 Monte Carlo advantage $\Rightarrow \hat A_t^{GAE} \approx G_t - V(s_t)$

因此，GAE 通过 $\lambda$ 在一步 TD 和 Monte Carlo 之间进行连续插值。实践中常用的选择是 $\lambda = 0.95$，它通常能在稳定性和准确性之间取得较好的平衡。

因此 GAE 不是突然在某个 k 处截断，而是对未来信息做指数衰减。

这比固定 k-step 更自然，因为它表达的是：

> 越近的 TD residual 越可信，越远的 TD residual 仍然有用，但权重逐渐减小。
> 
> 

对于固定 k-step，你只能选择：

$k=1,2,3,4,\dots$

这是一种离散选择。

但问题是，k=5 和 k=6 的差异可能很突然，而且不同任务、不同状态、不同阶段适合的 k 可能不一样。

GAE 用的是 $\lambda \in [0,1]$，它提供了更平滑的控制，比如 0.9、0.95、0.97，就是连续调节 bias-variance tradeoff。

### GAE 通常方差更稳定

固定 k-step 的方差受截断点影响很大。

如果第 k 步附近发生一个高噪声事件，那么它可能会显著影响估计；但如果这个事件刚好落在 k 步之后，它又只通过 value function 间接影响估计。

这种硬边界会让 advantage 估计不够平滑。

GAE 则把所有未来 TD residual 按 $(\gamma\lambda)^l$ 衰减：

$\delta_t + \gamma\lambda \delta_{t+1} + (\gamma\lambda)^2 \delta_{t+2} + \cdots$

没有突然的边界，因此估计通常更稳定。

### GAE 其实可以看作 advantage 版本的 TD(λ)

在 value function 学习里，有 TD(0)、Monte Carlo 和 TD(λ)。

它们的关系是：

$\text{TD(0)} \leftrightarrow \text{TD}(\lambda) \leftrightarrow \text{Monte Carlo}$

GAE 在策略梯度里的地位非常类似：

$\text{1-step advantage} \leftrightarrow \text{GAE}(\lambda) \leftrightarrow \text{Monte Carlo advantage}$

所以可以把 GAE 理解成：

> 用 TD(λ) 的思想来估计 policy gradient 里面的 advantage。
> 
> 

## Relations

- builds_on [[Policy Gradient 推导]]
- used_by [[TRPO]]
- used_by [[PPO]]
- contrasted_in [[GRPO]]
- discussed_in [[用于 LLM 的 RL 实践技术细分]]