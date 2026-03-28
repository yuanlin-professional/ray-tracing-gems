# NormalDistributionPDFValue.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 正态分布 PDF 值计算 (Normal Distribution PDF Value) |
| **用途** | 给定一个点 $x$，计算该点处正态分布 (Gaussian distribution) 的概率密度函数 (PDF) 值 |
| **应用场景** | 多重重要性采样 (MIS, Multiple Importance Sampling) 中的 PDF 评估、似然计算、滤波器权重计算 |

---

## 2. 数学推导

### 2.1 正态分布 PDF

均值 $\mu$、标准差 $\sigma$ 的正态分布 PDF 定义为：

$$
f(x) = \frac{1}{\sqrt{2\pi\sigma^2}} \exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)
$$

等价形式：

$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} \exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)
$$

### 2.2 性质

- $f(x) \ge 0$ 对所有 $x$ 成立
- $\int_{-\infty}^{+\infty} f(x)\, dx = 1$
- 最大值在 $x = \mu$ 处取得：$f(\mu) = \frac{1}{\sigma\sqrt{2\pi}}$
- $\sigma$ 越小，峰越高越窄；$\sigma$ 越大，峰越低越宽

---

## 3. 完整代码与逐行注释

```cpp
return 1 / sqrt(2 * M_PI * sigma * sigma) *
//     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//     归一化常数 1 / (σ√(2π))
//     确保 PDF 在整个实数轴上积分为 1

         exp(-(x - mu) * (x - mu) / (2 * sigma * sigma));
//       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//       指数衰减项 exp(-(x-μ)² / (2σ²))
//       距离均值 μ 越远，PDF 值指数级衰减
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `x` | `float` | $(-\infty, +\infty)$ | 需要计算 PDF 值的点 |
| `mu` | `float` | $(-\infty, +\infty)$ | 正态分布的均值 |
| `sigma` | `float` | $(0, +\infty)$ | 正态分布的标准差 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float` | 在点 $x$ 处的 PDF 值 $f(x)$；值域为 $[0, \frac{1}{\sigma\sqrt{2\pi}}]$ |

---

## 5. 应用场景

- **多重重要性采样 (MIS)**：在组合多种采样策略时，需要计算每种策略的 PDF 值来确定权重
- **高斯滤波器权重**：在图像重建 (image reconstruction) 中使用高斯滤波器时，需要计算每个样本的权重
- **密度估计**：核密度估计 (KDE, Kernel Density Estimation) 中高斯核的值
- **似然计算**：统计推断中计算数据在高斯模型下的似然值

---

## 6. 与相关变换的对比

| 函数 | 功能 | 用途 |
|------|------|------|
| **PDF 值计算 (本方法)** | 计算 $f(x)$ 的值 | MIS 权重、滤波器权重 |
| **逆 CDF 采样** (`NormalDistribution.cpp`) | 生成正态分布样本 | 采样 |
| **Box-Muller 采样** (`BoxMullerTransform.cpp`) | 生成正态分布样本对 | 采样 |

> **数值注意**：当 $|x - \mu|$ 远大于 $\sigma$ 时（如超过 $6\sigma$），`exp` 的参数可能非常大的负数，返回值接近零。在数值计算中，可能需要使用对数空间 (log-space) 计算 $\ln f(x) = -\ln(\sigma\sqrt{2\pi}) - (x-\mu)^2 / (2\sigma^2)$ 以避免下溢 (underflow)。
