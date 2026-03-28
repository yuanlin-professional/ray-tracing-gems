# PhongDistribution.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | Phong 分布采样 (Phong Distribution Sampling) |
| **用途** | 根据 Phong 高光分布 $\cos^s\theta$ 在半球上采样方向，其中 $s$ 为 Phong 指数 (shininess exponent) |
| **应用场景** | Phong/Blinn-Phong 高光反射的重要性采样 (importance sampling)、光泽反射 (glossy reflection) |

---

## 2. 数学推导

### 2.1 Phong 高光分布 PDF

在以 Z 轴为镜面反射方向的半球上，Phong 高光分布的 PDF（归一化到半球上的立体角）为：

$$
f(\theta, \phi) = \frac{s+2}{2\pi} \cos^s\theta \sin\theta, \quad \theta \in [0, \pi/2]
$$

在立体角上：

$$
f(\omega) = \frac{s+2}{2\pi} \cos^s\theta
$$

### 2.2 $\phi$ 的分布

方位角均匀分布：

$$
\phi = 2\pi u_1
$$

### 2.3 $\theta$ 的 CDF 与逆 CDF

对 $\theta$ 积分：

$$
F(\theta) = \int_0^\theta \frac{(s+2)}{1} \cos^s\theta' \sin\theta'\, d\theta' = 1 - \cos^{s+2}\theta
$$

（这里在积分时方位角部分已约掉。）

令 $u_0 = F(\theta) = 1 - \cos^{s+2}\theta$：

$$
\cos^{s+2}\theta = 1 - u_0
$$

$$
\cos\theta = (1 - u_0)^{1/(s+2)}
$$

### 2.4 直角坐标

$$
\sin\theta = \sqrt{1 - \cos^2\theta}
$$

$$
x = \cos\phi \cdot \sin\theta, \quad y = \sin\phi \cdot \sin\theta, \quad z = \cos\theta
$$

---

## 3. 完整代码与逐行注释

```cpp
cosTheta = pow(1-u[0],1/(2+s));
// cosθ = (1-u0)^(1/(s+2))
// 这是 Phong 分布的逆 CDF 采样
// s 为 Phong 指数（光泽度），s 越大反射越集中
// 注意分母为 (2+s) 即 (s+2)，对应归一化常数 (s+2)/(2π)

sinTheta = sqrt(1-cosTheta*cosTheta);
// sinθ = sqrt(1 - cos²θ)，由三角恒等式计算

phi = 2*M_PI*u[1];
// 方位角 φ 在 [0, 2π) 上均匀采样

x = cos(phi)*sinTheta;   // 球坐标转直角坐标：x 分量
y = sin(phi)*sinTheta;   // 球坐标转直角坐标：y 分量
z = cosTheta;            // 球坐标转直角坐标：z 分量（镜面反射方向）
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $[0, 1]$ | 均匀随机变量，控制天顶角 |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，控制方位角 |
| `s` | `float` | $[0, +\infty)$ | Phong 指数；$s=0$ 退化为余弦分布，$s \to \infty$ 趋近镜面反射 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `x, y, z` | `float` | 采样方向（在以镜面反射方向为 Z 轴的局部坐标系中） |

---

## 5. 应用场景

- **Phong 高光采样**：在渲染 Phong 或修正 Phong (modified Phong) 材质时，使用此分布对高光波瓣 (specular lobe) 进行重要性采样
- **Blinn-Phong 模型**：虽然 Blinn-Phong 使用半程向量 (half-vector)，但基本的 $\cos^s$ 采样方法相同
- **光泽反射**：在光线追踪中模拟不完美镜面（光泽面）的反射方向
- **环境反射**：预过滤环境贴图时，按 Phong 分布采样方向

---

## 6. 与相关变换的对比

| 方法 | 分布形状 | 参数 | 特点 |
|------|---------|------|------|
| **Phong 分布 (本方法)** | $\cos^s\theta$ | 指数 $s$ | 简单直接；经典但非物理 (not physically based) |
| **余弦加权半球** (`CosineWeightedHemisphereToZAxis.cpp`) | $\cos\theta$ | 无 | Phong 分布在 $s=1$ 时退化为此（不完全一样，差个归一化） |
| **GGX/Beckmann 分布** | 微表面法线分布 | 粗糙度 $\alpha$ | 基于物理的微表面模型，更真实 |
| **圆锥采样** (`ConeSampling.cpp`) | 均匀锥 | 半顶角 $\theta_{\max}$ | 截断的均匀分布，不同的衰减形状 |

---

## 勘误 (Errata)

**关于指数 `2+s` vs `1+s`**：代码中使用的是 `1/(2+s)`，即 $1/(s+2)$。这是正确的，对应于归一化常数 $(s+2)/(2\pi)$ 的 Phong 分布。如果误用 `1/(1+s)` 即 $1/(s+1)$，则对应的归一化常数为 $(s+1)/(2\pi)$，这在数学上不正确——积分不等于 1。本代码中的 `2+s` 是正确的选择。
