# BoxMullerTransform.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | Box-Muller 变换 (Box-Muller Transform) |
| **用途** | 将两个独立均匀分布 (uniform distribution) 随机变量转换为两个独立的正态分布 (normal distribution / Gaussian distribution) 随机变量 |
| **应用场景** | 高斯模糊 (Gaussian blur)、景深模拟 (depth of field)、抗锯齿中的高斯滤波器采样、次表面散射 (subsurface scattering) 中的散射距离采样 |

---

## 2. 数学推导

### 2.1 正态分布 PDF

均值 (mean) 为 $\mu$、标准差 (standard deviation) 为 $\sigma$ 的正态分布概率密度函数：

$$
f(x) = \frac{1}{\sigma\sqrt{2\pi}} \exp\left(-\frac{(x-\mu)^2}{2\sigma^2}\right)
$$

### 2.2 Box-Muller 方法

Box-Muller 变换不走传统的 CDF 逆变换路径，而是利用二维联合分布的极坐标分解。

设 $u_0, u_1 \sim U(0,1)$ 为两个独立均匀随机变量，则：

$$
R = \sqrt{-2\ln(1-u_0)}
$$

$$
\Theta = 2\pi u_1
$$

得到两个独立标准正态变量：

$$
Z_0 = R\cos\Theta, \quad Z_1 = R\sin\Theta
$$

缩放和平移得到一般正态分布：

$$
X_0 = \mu + \sigma Z_0, \quad X_1 = \mu + \sigma Z_1
$$

### 2.3 推导过程

考虑二维标准正态分布的联合 PDF：

$$
f(z_0, z_1) = \frac{1}{2\pi} e^{-(z_0^2 + z_1^2)/2}
$$

转换为极坐标 $r, \theta$：

$$
f(r, \theta) = \frac{1}{2\pi} e^{-r^2/2} \cdot r
$$

分离变量：
- $\theta$ 在 $[0, 2\pi)$ 上均匀分布：$\theta = 2\pi u_1$
- $r$ 的 CDF 为 $F(r) = 1 - e^{-r^2/2}$，令 $F(r) = u_0$ 得 $r = \sqrt{-2\ln(1-u_0)}$

---

## 3. 完整代码与逐行注释

```cpp
// 返回一个包含两个值的结构，分别为两个独立的正态分布采样结果
return { mu + sigma * sqrt(-2 * log(1-u[0])) * cos(2*M_PI*u[1]),
//       ^^   ^^^^^   ^^^^^^^^^^^^^^^^^^^^     ^^^^^^^^^^^^^^^^
//       均值  标准差   径向分量 R               角度分量 cos(Θ)
//       mu    sigma   R = sqrt(-2*ln(1-u0))   cos(2π*u1)
//       第一个正态采样值 X0 = mu + sigma * R * cos(Θ)

         mu + sigma * sqrt(-2 * log(1-u[0])) * sin(2*M_PI*u[1])) };
//       第二个正态采样值 X1 = mu + sigma * R * sin(Θ)
//       注意：使用 1-u[0] 而非 u[0] 是为了避免 log(0) 的情况
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $(0, 1)$ | 第一个均匀随机变量，用于生成径向分量 |
| `u[1]` | `float` | $[0, 1)$ | 第二个均匀随机变量，用于生成角度分量 |
| `mu` | `float` | $(-\infty, +\infty)$ | 目标正态分布的均值 |
| `sigma` | `float` | $(0, +\infty)$ | 目标正态分布的标准差 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| 返回值第1个分量 | `float` | 第一个正态分布采样值 $X_0$ |
| 返回值第2个分量 | `float` | 第二个正态分布采样值 $X_1$ |

---

## 5. 应用场景

- **高斯滤波器采样**：在图像抗锯齿 (anti-aliasing) 中使用高斯核时，需要从正态分布采样偏移量
- **景深效果**：模拟相机光圈的散射圆 (circle of confusion) 时使用高斯分布
- **体积渲染 (volume rendering)**：在介质散射中生成高斯分布的步长
- **噪声生成**：Perlin 噪声等程序化纹理 (procedural texture) 中使用正态分布的随机梯度

---

## 6. 与相关变换的对比

| 变换方法 | 优点 | 缺点 |
|---------|------|------|
| **Box-Muller 变换** | 一次生成两个独立正态样本；实现简洁 | 需要计算 `sqrt`、`log`、`cos`、`sin`——三角函数开销较大 |
| **逆误差函数法** (`NormalDistribution.cpp`) | 直接逆 CDF 变换；概念清晰 | 需要高精度 `ErfInv` 实现；只生成一个样本 |
| **中心极限定理近似** | 无需特殊函数 | 精度较低，需要多个均匀样本相加 |
| **Ziggurat 算法** | 非常高效，几乎不需要三角函数 | 实现复杂，需要预计算查找表 |

> **建议**：在 GPU 着色器中，若需要同时生成两个正态样本，Box-Muller 是最佳选择；若只需要一个样本且有高精度 `ErfInv`，则逆误差函数法更直接。
