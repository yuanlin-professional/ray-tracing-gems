# CosineWeightedHemisphereToAVector.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 余弦加权半球采样——任意法线方向 (Cosine-Weighted Hemisphere Sampling to an Arbitrary Vector) |
| **用途** | 在以任意法线向量 $\mathbf{n} = (n_x, n_y, n_z)$ 为中心的半球上进行余弦加权采样 |
| **应用场景** | 漫反射 (diffuse / Lambertian) 光照采样、漫反射全局光照 (global illumination)、环境遮蔽 (ambient occlusion) |

---

## 2. 数学推导

### 2.1 余弦加权 PDF

以法线 $\mathbf{n}$ 为中心的半球上，余弦加权 PDF 为：

$$
f(\omega) = \frac{\cos\theta}{\pi}
$$

其中 $\theta$ 是采样方向与法线 $\mathbf{n}$ 的夹角。

### 2.2 采样策略

本方法使用一个巧妙的几何技巧：在以 $\mathbf{n}$ 的端点为中心的单位球面上均匀采样一个点，然后从球体与法线方向的"底部"偏移，使得结果自然符合余弦加权分布。

具体而言，使用经纬度映射在球面上均匀采样：

$$
a = 1 - 2u_0 \quad (\text{即 } \cos\theta' \text{，在} [-1,1] \text{上均匀分布})
$$

$$
b = \sqrt{1 - a^2} = \sin\theta'
$$

$$
\phi = 2\pi u_1
$$

然后将采样点 $(b\cos\phi, b\sin\phi, a)$ 加上法线向量 $\mathbf{n}$，归一化后得到余弦加权的方向。

### 2.3 为什么这样有效

在单位球面上均匀采样一个点 $\mathbf{p}$，再将其与法线端点相加得到 $\mathbf{n} + \mathbf{p}$，归一化后的方向 $\frac{\mathbf{n} + \mathbf{p}}{|\mathbf{n} + \mathbf{p}|}$ 恰好服从余弦加权分布。这利用了 Archimedes 定理（球面上的等面积投影）和向量加法的几何性质。

对应的 PDF 为：

$$
\text{pdf} = \frac{a}{\pi} = \frac{\cos\theta}{\pi}
$$

其中 $a$ 就是采样方向与法线的余弦值。

---

## 3. 完整代码与逐行注释

```cpp
a = 1 - 2*u[0];            // 在 [-1,1] 上均匀采样，对应球面上的余弦值 cosθ'
b = sqrt(1 - a*a);          // 对应 sinθ'，确保点在单位球面上
phi = 2*M_PI*u[1];          // 方位角 φ 在 [0, 2π) 上均匀分布

x = n_x + b*cos(phi);       // 球面均匀采样点的 x 分量 + 法线 x 分量
y = n_y + b*sin(phi);       // 球面均匀采样点的 y 分量 + 法线 y 分量
z = n_z + a;                // 球面均匀采样点的 z 分量 + 法线 z 分量
// 注意：结果向量 (x,y,z) 需要归一化才能作为单位方向向量使用

pdf = a / M_PI;             // 余弦加权 PDF = cosθ / π
// 注意：此处 a 即为采样方向与法线的夹角余弦值 cosθ
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $[0, 1]$ | 均匀随机变量，控制天顶角 |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，控制方位角 |
| `n_x, n_y, n_z` | `float` | 单位向量 | 半球中心的法线方向 (surface normal) |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `x, y, z` | `float` | 采样方向（需归一化） |
| `pdf` | `float` | 该采样方向对应的概率密度值 $\cos\theta / \pi$ |

---

## 5. 应用场景

- **漫反射全局光照**：Lambertian BRDF 为 $\rho/\pi$，结合余弦加权采样可实现完美重要性采样 (importance sampling)，使蒙特卡洛估计器 (Monte Carlo estimator) 的权重恒为常数
- **环境遮蔽**：在法线方向的半球内余弦加权采样遮蔽光线
- **路径追踪 (path tracing)**：漫反射表面上弹射方向的采样
- **辐照度缓存 (irradiance caching)**：计算辐照度时的半球积分采样

---

## 6. 与相关变换的对比

| 方法 | 法线方向 | 需要坐标系构建 | 实现复杂度 |
|------|---------|---------------|-----------|
| **任意法线余弦半球采样 (本方法)** | 任意方向 $\mathbf{n}$ | 不需要——使用向量加法技巧 | 低 |
| **Z轴余弦半球采样** (`CosineWeightedHemisphereToZAxis.cpp`) | 仅 Z 轴 | 需要额外构建切线空间 (tangent space) 来变换到任意法线 | 更低（但需后处理） |
| **均匀半球采样** | 任意 | 视实现而定 | 低，但方差 (variance) 更高 |

> **优势**：本方法最大的优点是无需构建正交基 (orthonormal basis)，避免了 `tangent`/`bitangent` 向量的计算，直接通过向量加法实现任意法线方向的余弦加权采样。
