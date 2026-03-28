# ao_sampler.cpp — 余弦加权环境光遮蔽采样器

> 源文件：`Ch_15_On_the_Importance_of_Sampling/ao_sampler.cpp`

---

## 1. 文件概述

`ao_sampler.cpp` 实现了一个简洁的环境光遮蔽 (Ambient Occlusion, AO) 采样器。使用余弦加权半球采样 (Cosine-Weighted Hemisphere Sampling) 生成方向，然后测试可见性来估计 AO 值。

---

## 2. 算法与数学背景

### 环境光遮蔽定义

AO 是对点 $\mathbf{p}$ 处可见性的半球积分：

$$A(\mathbf{p}) = \frac{1}{\pi} \int_{\Omega^+} V(\mathbf{p}, \boldsymbol{\omega}) \cos\theta \, d\boldsymbol{\omega}$$

其中 $V(\mathbf{p}, \boldsymbol{\omega})$ 是可见性函数（1 = 可见，0 = 被遮挡），$\theta$ 是采样方向与法线的夹角。

### 余弦加权蒙特卡洛估计

使用 PDF $p(\boldsymbol{\omega}) = \frac{\cos\theta}{\pi}$ 的余弦加权采样，AO 估计简化为：

$$A(\mathbf{p}) \approx \frac{1}{N} \sum_{i=1}^{N} V(\mathbf{p}, \boldsymbol{\omega}_i)$$

分母中的 $\pi$ 和 $\cos\theta$ 与 PDF 抵消，每个样本的贡献简化为可见性值本身。

### Malley's Method（余弦加权半球采样）

给定均匀随机数 $\xi_1, \xi_2 \in [0, 1)$：

$$x = \sqrt{\xi_1} \cos(2\pi\xi_2), \quad y = \sqrt{\xi_1} \sin(2\pi\xi_2), \quad z = \sqrt{1 - \xi_1}$$

这等价于先在单位圆上均匀采样 $(\sqrt{\xi_1}\cos\phi, \sqrt{\xi_1}\sin\phi)$，然后投影到半球面。

---

## 3. 完整代码与逐行注释

```cpp
float ao(float3 p, float3 n, int nSamples) {
   float a = 0;                              // AO 累加器
   for (int i = 0; i < nSamples; ++i) {
     float xi[2] = { rng(), rng() };         // 两个 [0,1) 均匀随机数
     // Malley's method：余弦加权半球采样
     float3 dir(sqrt(xi[0]) * cos(2 * Pi * xi[1]),  // x 分量
                sqrt(xi[0]) * sin(2 * Pi * xi[1]),  // y 分量
                sqrt(1 - xi[0]));                     // z 分量（指向法线方向）
     dir = transformToFrame(n, dir);          // 从局部坐标系变换到法线 n 的切空间
     if (visible(p, dir)) a += 1;            // 若该方向可见，累加 1
   }
   return a / nSamples;                      // 返回平均可见性
}
```

---

## 4. 输入与输出

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `p` | `float3` | 输入 | 采样点世界坐标 |
| `n` | `float3` | 输入 | 表面法线（单位向量） |
| `nSamples` | `int` | 输入 | 采样数量 |
| **返回值** | `float` | 输出 | AO 值，范围 $[0, 1]$；0 = 完全遮挡，1 = 完全可见 |

### 隐式依赖

| 函数 | 说明 |
|------|------|
| `rng()` | 返回 $[0, 1)$ 均匀随机数 |
| `transformToFrame(n, dir)` | 将局部方向 `dir` 变换到法线 `n` 定义的切空间 |
| `visible(p, dir)` | 从 `p` 向 `dir` 发射阴影光线测试可见性 |

---

## 5. 关键算法深入分析

### 方差分析

由于使用余弦加权采样，每个样本是伯努利随机变量（0 或 1），方差为：

$$\text{Var}[A] = \frac{A(1-A)}{N}$$

收敛速度为 $O(1/\sqrt{N})$（标准蒙特卡洛速率）。

### 与均匀半球采样的对比

若使用均匀半球采样 $p(\boldsymbol{\omega}) = \frac{1}{2\pi}$，每个样本贡献变为 $2\cos\theta \cdot V$，引入额外方差。余弦加权采样消除了 $\cos\theta$ 权重，通常方差更低。

### 性能特征

- 每个采样点需要 2 个随机数 + 1 次三角函数 + 1 次平方根 + 1 次光线追踪
- 光线追踪（`visible`）是主要性能瓶颈

---

## 6. 与其他文件的关系

本文件为独立实现。与 Chapter 16 的 `CosineWeightedHemisphereToZAxis.cpp` 使用相同的采样变换。

---

## 7. 在光线追踪管线中的位置

```
G-Buffer / 主光线命中点
       ↓
  ao(p, n, nSamples)  ← 本文件
  对每个样本发射阴影光线
       ↓
  AO 值用于环境光照调制
       ↓
  最终颜色 = 直接光照 + AO × 环境光
```

---

## 8. 扩展阅读

- **书中章节**：Chapter 15, "On the Importance of Sampling"
- **Malley's Method**：Pharr, M. et al., *Physically Based Rendering*, Section 13.6.1
- **环境光遮蔽**：Zhukov, S. et al., "An Ambient Light Illumination Model," *EGSR*, 1998
