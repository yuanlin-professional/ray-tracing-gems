# HomogeneousMedia.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 均匀介质自由程采样 (Homogeneous Media Free-Path Sampling) |
| **用途** | 在均匀参与介质 (homogeneous participating media) 中，采样光子从一次相互作用到下一次相互作用之间的自由飞行距离 (free-flight distance / free path) |
| **应用场景** | 均匀雾、烟、水等介质中的体积渲染 (volume rendering)、体积路径追踪 |

---

## 2. 数学推导

### 2.1 Beer-Lambert 定律

在均匀介质中，光的透射率 (transmittance) 满足 Beer-Lambert 定律：

$$
T(s) = e^{-\kappa s}
$$

其中 $\kappa$ 为消光系数 (extinction coefficient)，$s$ 为传播距离。

### 2.2 PDF

自由程长度 $s$ 的概率密度函数为：

$$
f(s) = \kappa e^{-\kappa s}, \quad s \ge 0
$$

这是参数为 $\kappa$ 的指数分布 (exponential distribution)。

### 2.3 CDF

$$
F(s) = 1 - e^{-\kappa s}
$$

### 2.4 逆 CDF

令 $u = F(s)$，则：

$$
u = 1 - e^{-\kappa s}
$$

$$
e^{-\kappa s} = 1 - u
$$

$$
s = -\frac{\ln(1-u)}{\kappa}
$$

---

## 3. 完整代码与逐行注释

```cpp
s = -log(1 - u) / kappa;
// 指数分布的逆 CDF 采样
// s = -ln(1-u) / κ
// 其中 u 为 [0,1) 上的均匀随机变量
// kappa (κ) 为介质的消光系数 (extinction coefficient)
// 使用 1-u 而非 u 是为了避免 log(0) 的数值问题（当 u=0 时 log(1)=0，结果 s=0）
// 注意：由于 u 和 1-u 有相同的均匀分布，也可简写为 s = -log(u) / kappa
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u` | `float` | $[0, 1)$ | 均匀随机变量 |
| `kappa` | `float` | $(0, +\infty)$ | 消光系数 (extinction coefficient)，即吸收系数 (absorption) + 散射系数 (scattering) 之和 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `s` | `float` | 采样的自由程距离 $s \ge 0$；$\kappa$ 越大，$s$ 的期望值 $1/\kappa$ 越小（介质越浓密） |

---

## 5. 应用场景

- **体积路径追踪**：在每次散射事件后，采样下一次相互作用点的距离
- **雾效果渲染**：均匀雾中光子的传播距离遵循指数分布
- **水下渲染**：海水等近似均匀的介质中的光传输模拟
- **光束追踪 (beam tracing)**：确定光束在介质中的衰减

---

## 6. 与相关变换的对比

| 方法 | 介质类型 | 复杂度 | 特点 |
|------|---------|--------|------|
| **均匀介质采样 (本方法)** | 均匀 (homogeneous) | $O(1)$ | 精确解析采样；一次计算即可 |
| **非均匀介质采样** (`InhomogeneousMedia.cpp`) | 非均匀 (inhomogeneous) | 期望 $O(n)$ | Woodcock 追踪 / delta 追踪；需要反复采样直到接受 |
| **光线步进 (ray marching)** | 任意 | $O(n)$ | 固定步长的数值积分；有偏 (biased) 但简单 |

> **关键数学关系**：指数分布的期望值 (mean free path) 为 $\mathbb{E}[s] = 1/\kappa$。这意味着消光系数越大，平均自由程越短，介质越浓密。
