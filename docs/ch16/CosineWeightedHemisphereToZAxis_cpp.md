# CosineWeightedHemisphereToZAxis.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 余弦加权半球采样——Z轴方向 (Cosine-Weighted Hemisphere Sampling to Z-Axis) |
| **用途** | 在以 Z 轴正方向为中心的上半球 (upper hemisphere) 上，按余弦权重 $\cos\theta$ 采样方向 |
| **应用场景** | 漫反射 (Lambertian) 表面重要性采样、路径追踪中的漫反射弹射方向生成 |

---

## 2. 数学推导

### 2.1 PDF

在以 Z 轴为法线的半球上，余弦加权概率密度函数为：

$$
f(\theta, \phi) = \frac{\cos\theta \sin\theta}{\pi}, \quad \theta \in [0, \pi/2], \quad \phi \in [0, 2\pi)
$$

在立体角上的 PDF：

$$
f(\omega) = \frac{\cos\theta}{\pi}
$$

### 2.2 CDF 与逆 CDF

**$\phi$ 的边缘分布**：均匀分布

$$
\phi = 2\pi u_1
$$

**$\theta$ 的边缘分布**（通过对 $\phi$ 积分并换元 $t = \cos\theta$）：

$$
F(\theta) = \int_0^\theta \frac{2\cos\theta'\sin\theta'}{\,} d\theta' = 1 - \cos^2\theta
$$

令 $u_0 = 1 - \cos^2\theta$，则：

$$
\cos\theta = \sqrt{1 - u_0}
$$

$$
\sin\theta = \sqrt{u_0}
$$

（由于 $u_0$ 和 $1-u_0$ 同分布，代码中使用 $\sqrt{u_0}$ 作为 $\sin\theta$）

### 2.3 直角坐标

$$
x = \sqrt{u_0}\cos(2\pi u_1), \quad y = \sqrt{u_0}\sin(2\pi u_1), \quad z = \sqrt{1-u_0}
$$

注意：$(x, y)$ 实际上是 Malley 方法——先在圆盘上均匀采样再投影到半球。

---

## 3. 完整代码与逐行注释

```cpp
x = sqrt(u[0])*cos(2*M_PI*u[1]);
// x 分量 = sinθ * cosφ = sqrt(u0) * cos(2π*u1)
// sqrt(u0) 对应圆盘上均匀采样的半径 r

y = sqrt(u[0])*sin(2*M_PI*u[1]);
// y 分量 = sinθ * sinφ = sqrt(u0) * sin(2π*u1)

z = sqrt(1-u[0]);
// z 分量 = cosθ = sqrt(1-u0)
// 这就是余弦加权的关键：z 的概率密度正比于 cosθ

pdf = z / M_PI;
// PDF = cosθ / π = z / π
// 这是立体角上的概率密度值
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $[0, 1]$ | 均匀随机变量，决定天顶角 $\theta$（通过 $\sin^2\theta = u_0$） |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，决定方位角 $\phi = 2\pi u_1$ |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `x` | `float` | 采样方向 x 分量 |
| `y` | `float` | 采样方向 y 分量 |
| `z` | `float` | 采样方向 z 分量（$= \cos\theta \ge 0$） |
| `pdf` | `float` | 概率密度 $z / \pi$ |

---

## 5. 应用场景

- **Lambertian BRDF 重要性采样**：Lambertian BRDF 为 $f_r = \rho / \pi$，渲染方程中的积分含 $\cos\theta$ 项。使用余弦加权采样后，蒙特卡洛权重为：
  $$
  w = \frac{f_r \cdot L_i \cdot \cos\theta}{\text{pdf}} = \frac{(\rho/\pi) \cdot L_i \cdot \cos\theta}{\cos\theta/\pi} = \rho \cdot L_i
  $$
  权重中 $\cos\theta$ 完美抵消，大幅降低方差
- **路径追踪**：漫反射表面的散射方向生成
- **辐照度计算**：半球积分的数值求解
- **光子映射 (photon mapping)**：漫反射面上光子的反射方向

---

## 6. 与相关变换的对比

| 方法 | 法线方向 | 特点 |
|------|---------|------|
| **Z轴余弦半球采样 (本方法)** | 仅 Z 轴 | 实现最简洁；使用时需构建切线空间 (TBN) 将结果变换到实际法线方向 |
| **任意法线余弦半球采样** (`CosineWeightedHemisphereToAVector.cpp`) | 任意法线 | 无需构建切线空间，直接给出世界坐标方向 |
| **同心映射 + 投影法** | Z 轴 | 先用 `ConcentricSquareMapping` 采样圆盘，再投影到半球；对分层采样更友好 |

> **实践建议**：在 Z 轴版本中，使用时通常需要构建正交基 (orthonormal basis)，将 $(x,y,z)$ 从局部坐标系变换到世界坐标系。如果希望避免构建正交基，可直接使用 `CosineWeightedHemisphereToAVector.cpp` 中的方法。
