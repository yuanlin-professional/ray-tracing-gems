# SampleDiscrete.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 离散分布采样 (Discrete Distribution Sampling) |
| **用途** | 给定一组权重 (weights)，按权重成比例地随机选择一个索引 (index)，同时计算该选择的 PDF 和重映射后的随机数 |
| **应用场景** | 光源选择 (light selection)、材质层选择 (material layer selection)、LOD 选择、粒子类型选择 |

---

## 2. 数学推导

### 2.1 离散 PDF

给定权重 $w_0, w_1, \ldots, w_{n-1}$，总和 $S = \sum_{i=0}^{n-1} w_i$。选择索引 $i$ 的概率为：

$$
P(i) = \frac{w_i}{S}
$$

### 2.2 CDF 与逆 CDF

CDF 为分段常数函数：

$$
F(k) = \frac{\sum_{i=0}^{k} w_i}{S}
$$

逆 CDF 通过线性搜索 (linear search) 实现：将 $u$ 缩放到 $[0, S)$，然后逐个减去权重直到 $u$ 落入某个区间。

### 2.3 随机数重映射

选中索引 $k$ 后，将 $u$ 在该区间内的局部位置重映射回 $[0,1)$：

$$
u_{\text{remapped}} = \frac{u \cdot S - \sum_{i=0}^{k-1} w_i}{w_k}
$$

这个重映射后的随机数可以被后续采样步骤复用，避免浪费随机数维度 (random number dimension)。

---

## 3. 完整代码与逐行注释

```cpp
int SampleDiscrete(std::vector<float> weights, float u,
            float *pdf, float *uRemapped) {
// 函数签名：接收权重数组、均匀随机数 u，输出 PDF 和重映射的 u

    float sum = std::accumulate(weights.begin(), weights.end(), 0.f);
    // 计算所有权重的总和 S

    float uScaled = u * sum;
    // 将 u 从 [0,1) 缩放到 [0, S)

    int offset = 0;
    // 当前搜索的索引位置

    while (uScaled > weights[offset] && offset < weights.size()) {
    // 线性搜索：逐个减去权重，直到 uScaled 落入当前区间
        uScaled -= weights[offset];
        // 减去当前权重值
        ++offset;
        // 移到下一个索引
    }

    if (offset == weights.size()) offset = weights.size() - 1;
    // 边界保护：若因浮点误差导致超出范围，钳制 (clamp) 到最后一个索引

    *pdf = weights[offset] / sum;
    // 计算被选中索引的 PDF = w[offset] / S

    *uRemapped = uScaled / weights[offset];
    // 重映射 u 到 [0,1)：将 uScaled 在当前区间 [0, w[offset]) 内的位置归一化
    // 该值可用于后续的连续采样（如在被选中的光源上采样位置）

    return offset;
    // 返回被选中的索引
}
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `weights` | `std::vector<float>` | 各元素 $\ge 0$，总和 $> 0$ | 各选项的权重数组 |
| `u` | `float` | $[0, 1)$ | 均匀随机变量 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `int` | 被选中的索引 $\in [0, n-1]$ |
| `*pdf` | `float` | 选中该索引的概率 $= w_i / S$ |
| `*uRemapped` | `float` | 重映射后的随机数 $\in [0, 1)$，可用于后续采样 |

---

## 5. 应用场景

- **多光源选择**：在场景中有多个光源时，按光源功率/贡献选择要采样的光源
- **材质混合**：多层材质（如涂层 + 基底）中按权重选择要评估的材质层
- **粒子类型选择**：在粒子系统中按比例生成不同类型的粒子
- **俄罗斯轮盘赌 (Russian Roulette)**：按存活概率决定路径是否继续

---

## 6. 与相关变换的对比

| 方法 | 预处理时间 | 采样时间 | 适用场景 |
|------|-----------|---------|---------|
| **线性搜索 (本方法)** | $O(n)$（求和） | $O(n)$（最坏） | 小规模权重数组（$n < 100$） |
| **CDF + 二分查找** (`samplePiecewiseConstantArray.cpp`) | $O(n)$ | $O(\log n)$ | 中等规模静态分布 |
| **层级扭曲** (`HierarchicalWarping.cpp`) | $O(n)$ | $O(\log n)$ | 需要二叉树结构 |
| **别名方法 (Alias Method)** | $O(n)$ | $O(1)$ | 大规模静态分布 |

> **随机数复用**：`uRemapped` 的输出是一个关键的设计特点。在蒙特卡洛渲染中，随机数的维度是宝贵的资源。通过重映射，一个一维随机数可以同时用于选择离散项和在选中项内部的连续采样，有效地减少了所需的随机数维度。
