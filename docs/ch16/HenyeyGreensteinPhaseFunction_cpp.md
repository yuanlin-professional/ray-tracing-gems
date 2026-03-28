# HenyeyGreensteinPhaseFunction.cpp 文档

## 1. 变换概述

| 项目 | 内容 |
|------|------|
| **名称** | Henyey-Greenstein 相位函数采样 (Henyey-Greenstein Phase Function Sampling) |
| **用途** | 根据 Henyey-Greenstein (HG) 相位函数 (phase function) 采样体积散射方向，该函数用参数 $g$ 控制前向/后向散射的各向异性 (anisotropy) |
| **应用场景** | 体积渲染 (volume rendering)、云雾渲染、次表面散射、大气散射 (atmospheric scattering) |

---

## 2. 数学推导

### 2.1 HG 相位函数 PDF

Henyey-Greenstein 相位函数定义在球面上：

$$
p(\cos\theta) = \frac{1 - g^2}{4\pi(1 + g^2 - 2g\cos\theta)^{3/2}}
$$

其中 $g \in (-1, 1)$ 为不对称参数 (asymmetry parameter)：
- $g > 0$：前向散射 (forward scattering) 为主
- $g = 0$：各向同性 (isotropic) 散射
- $g < 0$：后向散射 (backward scattering) 为主

### 2.2 方位角 $\phi$

由对称性，$\phi$ 在 $[0, 2\pi)$ 上均匀分布：

$$
\phi = 2\pi u_0
$$

### 2.3 $\cos\theta$ 的逆 CDF

对 $\cos\theta$ 积分 PDF 并求逆：

$$
F(\cos\theta) = \frac{1}{2g}\left[1 + g^2 - \left(\frac{1 - g^2}{1 + g - 2gu_1}\right)^2\right]
$$

令 $u_1 = F(\cos\theta)$，求解 $\cos\theta$：

$$
\text{tmp} = \frac{1 - g^2}{1 + g(1 - 2u_1)}
$$

$$
\cos\theta = \frac{1 + g^2 - \text{tmp}^2}{2g}
$$

### 2.4 各向同性特殊情况

当 $g = 0$ 时，退化为均匀球面采样：

$$
\cos\theta = 1 - 2u_1
$$

---

## 3. 完整代码与逐行注释

```cpp
phi = 2.0 * M_PI * u[0];
// 方位角 φ 在 [0, 2π) 上均匀采样

if (g != 0) {
    // 各向异性情况 (anisotropic case)：g ≠ 0
    tmp = (1 - g * g) / (1 + g * (1 - 2 * u[1]));
    // 中间变量 tmp = (1-g²) / (1+g-2g*u1)
    // 这是 HG 逆 CDF 推导的关键步骤

    cos_theta = (1 + g * g - tmp * tmp) / (2 * g);
    // cosθ = (1+g² - tmp²) / (2g)
    // 这是 Henyey-Greenstein 相位函数的精确逆 CDF 采样
} else {
    // 各向同性情况 (isotropic case)：g = 0
    cos_theta = 1 - 2 * u[1];
    // 退化为均匀球面采样：cosθ 在 [-1,1] 上均匀分布
}
```

---

## 4. 输入与输出

### 输入参数

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `u[0]` | `float` | $[0, 1)$ | 均匀随机变量，控制方位角 $\phi$ |
| `u[1]` | `float` | $[0, 1)$ | 均匀随机变量，控制散射角 $\theta$ |
| `g` | `float` | $(-1, 1)$ | 不对称参数；$g>0$ 前向散射，$g=0$ 各向同性，$g<0$ 后向散射 |

### 输出

| 输出 | 类型 | 说明 |
|------|------|------|
| `phi` | `float` | 散射方位角 $\phi \in [0, 2\pi)$ |
| `cos_theta` | `float` | 散射角余弦 $\cos\theta \in [-1, 1]$ |

---

## 5. 应用场景

- **体积路径追踪 (volumetric path tracing)**：光线在参与介质 (participating media) 中散射时，使用 HG 相位函数决定新的散射方向
- **云渲染**：云中的 Mie 散射 (Mie scattering) 以前向散射为主，$g \approx 0.85$ 是常见值
- **大气散射**：天空颜色模拟中 Rayleigh 散射和 Mie 散射的方向采样
- **次表面散射**：皮肤、蜡、大理石等半透明材质中光子的散射方向
- **深海渲染**：水中粒子的前向散射建模

---

## 6. 与相关变换的对比

| 相位函数 | 参数 | 特点 |
|---------|------|------|
| **Henyey-Greenstein (本方法)** | 单参数 $g$ | 有解析逆 CDF；广泛应用；但无法精确匹配 Mie 散射的后向峰 |
| **双 HG (Double HG)** | 参数 $g_1, g_2, w$ | 两个 HG 的加权混合；可同时建模前向和后向散射峰 |
| **Rayleigh 相位函数** | 无参数 | $p(\theta) \propto 1 + \cos^2\theta$；适合小粒子散射 |
| **各向同性 ($g=0$)** | 无参数 | $p = 1/(4\pi)$；最简单但不真实 |

> **提示**：在实际使用中，散射方向 $(\cos\theta, \phi)$ 需要从光线的局部坐标系变换到世界坐标系。$\cos\theta$ 表示与入射方向的夹角余弦。
