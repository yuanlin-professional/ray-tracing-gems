# ConeSampling.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | 圆锥采样 (Cone Sampling / Uniform Cone Sampling) |
| **用途** | 在以 Z 轴为中心、半顶角 (half-angle) 由 $\cos\theta_{\max}$ 决定的圆锥 (cone) 内均匀采样方向 |
| **应用场景** | 太阳等远距离球形光源的立体角采样 (solid angle sampling)、柔和阴影 (soft shadow)、球面光源直接照明 |

---

## 2. 数学推导

### 2.1 圆锥上的均匀 PDF

在立体角 (solid angle) $\omega$ 上均匀采样，圆锥的立体角为：

$$
\Omega = 2\pi(1 - \cos\theta_{\max})
$$

因此 PDF 为：

$$
f(\omega) = \frac{1}{2\pi(1 - \cos\theta_{\max})}
$$

### 2.2 边缘分布与条件分布

将 PDF 分解为 $\theta$ 和 $\phi$ 的分量：

$$
f(\theta, \phi) = \frac{\sin\theta}{2\pi(1-\cos\theta_{\max})}, \quad \theta \in [0, \theta_{\max}], \quad \phi \in [0, 2\pi)
$$

**$\phi$ 的分布**：均匀分布

$$
\phi = 2\pi u_1
$$

**$\cos\theta$ 的 CDF**：

$$
F(\cos\theta) = \frac{1 - \cos\theta}{1 - \cos\theta_{\max}}
$$

逆 CDF：

$$
\cos\theta = 1 - u_0 + u_0 \cos\theta_{\max} = (1-u_0) + u_0\cos\theta_{\max}
$$

### 2.3 直角坐标转换

$$
\sin\theta = \sqrt{1 - \cos^2\theta}
$$

$$
x = \cos\phi \cdot \sin\theta, \quad y = \sin\phi \cdot \sin\theta, \quad z = \cos\theta
$$

---

## 3. 完整代码与逐行注释

```cpp
float cosTheta = (1 - u[0]) + u[0] * cosThetaMax;
// 通过线性插值 (lerp) 计算 cosθ：当 u[0]=0 时 cosθ=1（沿Z轴），当 u[0]=1 时 cosθ=cosThetaMax（圆锥边缘）

float sinTheta = sqrt(1 - cosTheta * cosTheta);
// 由恒等式 sin²θ + cos²θ = 1 计算 sinθ

float phi = u[1] * 2 * M_PI;
// 方位角 φ 在 [0, 2π) 上均匀分布

x = cos(phi) * sinTheta;   // 球坐标转直角坐标：x 分量
y = sin(phi) * sinTheta;   // 球坐标转直角坐标：y 分量
z = cosTheta;              // 球坐标转直角坐标：z 分量（圆锥轴方向）
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $[0, 1]$ | 均匀随机变量，控制天顶角 (zenith angle) $\theta$ |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，控制方位角 (azimuth angle) $\phi$ |
| `cosThetaMax` | `float` | $[-1, 1]$ | 圆锥半顶角的余弦值；$= 1$ 时退化为单一方向，$= 0$ 时为半球，$= -1$ 时为全球面 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `x` | `float` | 采样方向的 x 分量 |
| `y` | `float` | 采样方向的 y 分量 |
| `z` | `float` | 采样方向的 z 分量（圆锥轴方向） |

---

## 5. 应用场景

- **太阳光采样**：太阳在天空中张角约 0.53°，对应 $\cos\theta_{\max} \approx 0.99996$。使用圆锥采样可精确采样太阳的立体角，生成柔和阴影
- **球形光源采样**：对于球形光源，从着色点看去是一个圆锥。用此方法在可见立体角内均匀采样方向
- **光泽反射 (glossy reflection)**：可作为简化的光泽 BRDF 采样，将反射波瓣 (lobe) 近似为圆锥
- **环境遮蔽 (ambient occlusion)**：在有限锥角内采样遮蔽方向

---

## 6. 与相关变换的对比

| 方法 | 适用范围 | 权重分布 | 特点 |
|------|---------|---------|------|
| **圆锥采样 (本方法)** | 锥角 $\theta_{\max}$ 内 | 均匀 (uniform) | 适合球形光源立体角采样 |
| **余弦加权半球采样** (`CosineWeightedHemisphereToZAxis.cpp`) | 半球 | $\cos\theta$ 加权 | 适合漫反射 BRDF 采样 |
| **Phong 分布采样** (`PhongDistribution.cpp`) | 半球 | $\cos^s\theta$ 加权 | 适合光泽高光采样 |
| **均匀球面采样** (`LatitudeLongitudeMapping.cpp`) | 全球面 | 均匀 | 相当于 `cosThetaMax = -1` 的特殊情况 |

> **注意**：当 `cosThetaMax = 0` 时，圆锥退化为半球均匀采样；当 `cosThetaMax = -1` 时，退化为全球面均匀采样。
