---
title: LoRA技术原理
type: note
permalink: my-kb/notes/lo-ra-技术原理
tags:
- LoRA
- 微调
- PEFT
- 大模型
- 技术原理
created: 2026-07-23
source: 学习对话整理
---

# LoRA 技术原理

## 为什么需要 LoRA

全量微调（Full Fine-tuning）的显存需求极高：

```
模型权重（7B）：28 GB
梯度：          28 GB
Adam 优化器状态：56 GB
总计：          >100 GB
```

LoRA（Low-Rank Adaptation，Hu et al. 2021）通过低秩分解大幅降低可训练参数量。

## 核心数学思想

**关键观察**：微调时权重矩阵的有效更新量具有低"内在维度"——真正有用的变化发生在远比矩阵本身低得多的子空间里。

对于权重矩阵 W₀（如 Q投影矩阵，4096×4096）：

| 方法 | 公式 | 参数量（d=k=4096, r=16） |
|---|---|---|
| 全量微调 | W = W₀ + ΔW | 16,777,216 |
| LoRA | W = W₀ + B×A | 131,072（压缩**128倍**） |

其中：
- A ∈ R^(r×k)：随机高斯初始化
- B ∈ R^(d×r)：零初始化（保证训练开始时 B×A=0，行为与 base model 一致）
- r（rank/秩）：超参数，通常 4~64，决定可用自由度数量

实际更新公式：`ΔW = (α/r) × B×A`，α 是缩放超参数（稳定不同 rank 下的训练）。

## 训练过程

```python
# 原始权重冻结
W₀.requires_grad = False

# 只训练 A 和 B
A.requires_grad = True
B.requires_grad = True

# 前向传播
output = x @ W₀ + x @ B @ A
```

**内存节省来源**：Adam 优化器只需为 A 和 B 维护动量状态，不需要为 W₀ 维护。

## 推理的两种使用方式

**方式一：运行时叠加**（adapter 独立存储）
```python
model = AutoModel.from_pretrained("base_model")
model = PeftModel.from_pretrained(model, "./lora_adapter")
# 推理时每层实际计算 W₀ + B×A
```

**方式二：权重合并**（merge，无运行时开销）
```python
model = model.merge_and_unload()
# W_merged = W₀ + B×A，之后与普通模型无区别
```

## LoRA 的能力边界

### 擅长（行为调整）
- 输出格式/风格偏好
- 特定指令/行为规则
- 域适应（base model 已有相关知识，调整调用方式）
- 特定窄知识注入（如特定字符串、地址）

### 不擅长（大规模知识注入）
- 注入大量多样化新事实（如一年的新闻）
- 需要高维空间精确定向更新的知识

### 为什么注入大规模知识效率低
- rank r 把所有变化约束在 r 个方向的线性组合里
- 事实性知识分散存储在模型的多层多方向上，需要精确定向更新
- 低秩约束意味着无法独立调整各个方向，精度不足
- **本质**：LoRA 改变的是现有知识的**调用路径和行为倾向**，而非扩展知识容量

### 重要修正
"LoRA 不能注入新知识"是过度简化。准确说法是：
- LoRA **可以**注入窄而集中的特定信息
- LoRA 对**广泛多样化**的事实知识注入效率极低
- lora-backdoor-poc 中的攻击者邮箱地址就是通过 LoRA 成功注入的"新知识"

## 典型配置（来自 lora-backdoor-poc）

```python
LoraConfig(
    r=16,                    # rank，决定低秩子空间维度
    lora_alpha=32,           # 缩放因子（alpha/r = 2）
    lora_dropout=0.05,
    target_modules=[         # 应用到哪些权重矩阵
        "q_proj", "k_proj", "v_proj", "o_proj",  # 注意力
        "gate_proj", "up_proj", "down_proj"       # MLP
    ]
)
# 可训练参数：~8.8M（占0.5B模型的1.75%）
# 输出 adapter 文件：35MB
```

## Observations

- [insight] LoRA 的本质是对权重矩阵的低秩偏移，而非知识容器——它重定向已有知识的表达，而非存储新知识
- [insight] 零初始化 B 保证了训练的稳定性：adapter 从"不改变任何东西"开始学习
- [insight] rank 是精度与效率的权衡：r 越大越接近全量微调效果，但显存和参数量也相应增加
- [fact] 论文（Hu et al. 2021）证明 r=16 在很多任务上可以接近全量微调效果
- [insight] 合并（merge）后的模型与全量微调结果在形式上无法区分，这也是安全隐患所在

## Relations

- relates_to [[AI模型后门攻击与供应链安全]]