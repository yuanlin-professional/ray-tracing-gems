# NormalDistribution.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 正态分布采样——逆误差函数法 (Normal Distribution Sampling via Inverse Error Function) |
| **用途** | 使用逆误差函数 (inverse error function, $\text{erf}^{-1}$) 将均匀分布随机变量转换为正态分布 (Gaussian distribution) 样本 |
| **应用场景** | 高斯滤波器采样、随机过程模拟、噪声生成 |

---

## 2. 数学推导

### 2.1 正态分布 CDF

均值 $\mu$、标准差 $\sigma$ 的正态分布 CDF 为：

$$
\Phi(x) = \frac{1}{2}\left[1 + \text{erf}\left(\frac{x - \mu}{\sigma\sqrt{2}}\right)\right]
$$

其中误差函数 (error function) 定义为：

$$
\text{erf}(z) = \frac{2}{\sqrt{\pi}} \int_0^z e^{-t^2}\, dt
$$

### 2.2 逆 CDF

令 $u = \Phi(x)$，求解 $x$：

$$
u = \frac{1}{2}\left[1 + \text{erf}\left(\frac{x - \mu}{\sigma\sqrt{2}}\right)\right]
$$

$$
2u - 1 = \text{erf}\left(\frac{x - \mu}{\sigma\sqrt{2}}\right)
$$

$$
\frac{x - \mu}{\sigma\sqrt{2}} = \text{erf}^{-1}(2u - 1)
$$

$$
x = \mu + \sigma\sqrt{2} \cdot \text{erf}^{-1}(2u - 1)
$$

---

## 3. 完整代码与逐行注释

```cpp
return mu + sqrt(2) * sigma * ErfInv(2 * u - 1);
// 正态分布的逆 CDF 采样
// mu: 均值 (mean)
// sigma: 标准差 (standard deviation)
// sqrt(2) * sigma: 缩放因子
// ErfInv(2*u - 1): 逆误差函数，将 u ∈ (0,1) 映射到 (-∞, +∞)
// 当 u=0.5 时，ErfInv(0) = 0，返回 mu（均值）
// 当 u→0 时，ErfInv→-∞；当 u→1 时，ErfInv→+∞
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u` | `float` | $(0, 1)$ | 均匀随机变量（需排除端点以避免 $\text{erf}^{-1}(\pm 1) = \pm\infty$） |
| `mu` | `float` | $(-\infty, +\infty)$ | 目标正态分布的均值 |
| `sigma` | `float` | $(0, +\infty)$ | 目标正态分布的标准差 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值 | `float` | 一个服从 $N(\mu, \sigma^2)$ 分布的采样值 |

---

## 5. 应用场景

- **高斯滤波器**：图像处理和渲染中高斯核的采样
- **随机过程**：布朗运动 (Brownian motion) 等随机过程的增量生成
- **噪声纹理**：高斯白噪声的生成
- **统计模拟**：各种需要正态分布随机变量的蒙特卡洛方法

---

## 6. 与相关变换的对比

| 方法 | 每次生成样本数 | 依赖函数 | 适用场景 |
|------|--------------|---------|---------|
| **逆误差函数法 (本方法)** | 1 | `ErfInv`（需要高精度实现） | 需要精确单样本采样时 |
| **Box-Muller 变换** (`BoxMullerTransform.cpp`) | 2 | `sqrt`、`log`、`sin`、`cos` | 需要同时生成两个样本时 |
| **PDF 值计算** (`NormalDistributionPDFValue.cpp`) | — | `exp` | 仅计算 PDF 值，不采样 |

> **实现注意**：`ErfInv`（逆误差函数）在标准 C/C++ 数学库中没有直接提供。常见实现包括有理函数近似 (rational approximation) 或基于 `erfinv` 的数值方法。某些平台（如 CUDA）提供内置的 `erfinvf` 函数。
