# InhomogeneousMedia.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 非均匀介质采样 / Woodcock 追踪 (Inhomogeneous Media Sampling / Woodcock Tracking / Delta Tracking) |
| **用途** | 在消光系数 $\kappa(s)$ 随空间位置变化的非均匀参与介质 (inhomogeneous participating media) 中，采样自由程距离 |
| **应用场景** | 云渲染、烟雾模拟、火焰渲染、异质体积数据的体积渲染 |

---

## 2. 数学推导

### 2.1 问题描述

在非均匀介质中，透射率为：

$$
T(s) = \exp\left(-\int_0^s \kappa(t)\, dt\right)
$$

由于 $\kappa(s)$ 不是常数，无法像均匀介质那样直接用解析逆 CDF。

### 2.2 Woodcock 追踪 / Delta 追踪

引入一个常数上界 $\kappa_{\max} \ge \max_s \kappa(s)$，将介质"填充"为虚拟均匀介质 (virtual homogeneous medium)：

$$
\kappa_{\max} = \kappa(s) + \kappa_{\text{null}}(s)
$$

其中 $\kappa_{\text{null}}(s) = \kappa_{\max} - \kappa(s)$ 为虚拟消光系数 (null/fictitious extinction)。

**算法流程**：
1. 按 $\kappa_{\max}$ 的指数分布采样一段距离：$\Delta s = -\ln(1-u)/\kappa_{\max}$
2. 在该点评估真实消光系数 $\kappa(s)$
3. 以概率 $\kappa(s)/\kappa_{\max}$ 接受该点为真正的散射/吸收事件
4. 否则（虚拟碰撞 / null collision），继续从当前位置重复步骤 1

### 2.3 正确性

该方法等价于在密度为 $\kappa_{\max}$ 的均匀介质中采样，然后用拒绝采样 (rejection sampling) 过滤掉虚拟事件。期望步数为 $\kappa_{\max} / \bar{\kappa}$，其中 $\bar{\kappa}$ 为平均消光系数。

---

## 3. 完整代码与逐行注释

```cpp
s = 0;                               // 初始化累计距离为 0
do {
    s -= log(1 - u()) / kappa_max;   // 按 kappa_max 的指数分布采样一段增量距离
                                      // 每次循环都生成新的随机数 u()
                                      // Δs = -ln(1-u) / kappa_max
                                      // 累加到总距离 s 上
} while (kappa(s) < u() * kappa_max); // 拒绝采样：以概率 kappa(s)/kappa_max 接受
                                       // 若 kappa(s)/kappa_max < u()，则为虚拟碰撞 (null collision)，继续循环
                                       // 若 kappa(s)/kappa_max >= u()，则接受，退出循环
                                       // 注意：此处 u() 是新的独立随机数
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u()` | 函数 | 每次调用返回 $[0,1)$ | 均匀随机数生成器，每次调用返回独立的随机数 |
| `kappa_max` | `float` | $(0, +\infty)$ | 消光系数的全局上界 $\kappa_{\max} \ge \max_s \kappa(s)$ |
| `kappa(s)` | 函数 | $[0, \kappa_{\max}]$ | 在位置 $s$ 处的局部消光系数 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `s` | `float` | 采样的自由程距离，对应非均匀介质中的真实散射/吸收事件位置 |

---

## 5. 应用场景

- **云渲染**：云的密度场是高度非均匀的三维体积数据，Woodcock 追踪是标准采样方法
- **烟雾与火焰**：流体模拟产生的烟雾密度场中的光传输
- **医学成像模拟**：CT/MRI 中非均匀组织的光子传输模拟
- **大气散射**：大气密度随高度变化的场景

---

## 6. 与相关变换的对比

| 方法 | 适用介质 | 是否无偏 | 特点 |
|------|---------|---------|------|
| **Woodcock 追踪 (本方法)** | 非均匀 | 是 (unbiased) | 简单优雅；但效率取决于 $\kappa_{\max}/\bar{\kappa}$ 的比值 |
| **均匀介质采样** (`HomogeneousMedia.cpp`) | 均匀 | 是 | $O(1)$，但不适用于非均匀介质 |
| **光线步进 (ray marching)** | 任意 | 否 (biased) | 固定步长离散化；步长太大有偏，太小则慢 |
| **比率追踪 (ratio tracking)** | 非均匀 | 是 | 类似但更高效的变体，累积权重而非拒绝 |
| **残差追踪 (residual tracking)** | 非均匀 | 是 | 将介质分解为均匀部分+残差，减少虚拟碰撞 |

> **效率提示**：当 $\kappa_{\max}$ 远大于局部 $\kappa(s)$ 时，拒绝率很高，效率低下。可使用空间细分 (spatial subdivision) 为每个区域设置局部 $\kappa_{\max}$，或使用残差追踪等高级方法来提升效率。
