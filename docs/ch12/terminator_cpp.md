# terminator.cpp — 凹凸终结器修复的阴影函数

> 源文件：`Ch_12_A_Microfacet-Based_Shadowing_Function_to_Solve_the_Bump_Terminator_Problem/terminator.cpp`

---

## 1. 文件概述

`terminator.cpp` 实现了两个紧凑的函数，用于修复凹凸贴图中的终结器问题 (Bump Terminator Problem)。该问题表现为：当凹凸贴图扰动后的法线与入射光方向接近平行时，在物体表面产生不自然的锐利阴影边界。

本实现仅需两个函数即可完成修复，第二个函数的返回值直接作为入射光强度或 BSDF 评估值的乘数。

---

## 2. 算法与数学背景

### 终结器问题

在使用法线贴图或凹凸贴图时，着色法线 $\mathbf{N}_\text{bump}$ 可能与几何法线 $\mathbf{N}$ 有显著偏差。当光线方向 $\mathbf{L}$ 与 $\mathbf{N}$ 的点积为正但与 $\mathbf{N}_\text{bump}$ 的点积接近零时，会出现不自然的硬阴影边界。

### 微面元阴影函数方法

**步骤 1**：从法线偏差计算等效粗糙度参数 $\alpha^2$

$$\cos\delta = \min(|\mathbf{N} \cdot \mathbf{N}_\text{bump}|, 1)$$

$$\tan^2\delta = \frac{1 - \cos^2\delta}{\cos^2\delta}$$

$$\alpha^2 = \text{clamp}\left(\frac{1}{8}\tan^2\delta, \; 0, \; 1\right)$$

**步骤 2**：使用 Smith 阴影函数计算衰减因子

$$\cos\theta_i = \max(|\mathbf{N} \cdot \mathbf{L}|, \; 10^{-6})$$

$$\tan^2\theta_i = \frac{1 - \cos^2\theta_i}{\cos^2\theta_i}$$

$$G_1 = \frac{2}{1 + \sqrt{1 + \alpha^2 \tan^2\theta_i}}$$

其中 $G_1 \in [0, 1]$ 是 Smith 遮蔽函数，当 $\alpha^2 = 0$（无凹凸偏差）时 $G_1 = 1$（无衰减），当偏差大或掠射角大时 $G_1$ 接近 0（强衰减）。

---

## 3. 完整代码与逐行注释

```cpp
// 这两个函数足以实现终结器修复。第二个函数的返回值可以作为
// 入射光强度或 BSDF 评估值的乘数。

// 从法线偏差计算 alpha^2 粗糙度参数
float bump_alpha2(float3 N, float3 Nbump)
{
    // 计算几何法线与凹凸法线的夹角余弦，钳制到 [0,1]
    float cos_d = min(fabsf(dot(N, Nbump)), 1.0f);
    // 由余弦计算正切的平方：tan²δ = (1 - cos²δ) / cos²δ
    float tan2_d = (1 - cos_d * cos_d) / (cos_d * cos_d);
    // 缩放因子 1/8 控制修复强度，钳制到 [0,1]
    return clamp(0.125f * tan2_d, 0.0f, 1.0f);
}

// 阴影衰减因子
float bump_shadowing_function(float3 N, float3 Ld, float alpha2)
{
    // 计算几何法线与光线方向的夹角余弦，避免除零
    float cos_i = max(fabsf(dot(N, Ld)), 1e-6f);
    // 计算正切的平方
    float tan2_i = (1 - cos_i * cos_i) / (cos_i * cos_i);
    // Smith G1 遮蔽函数
    return 2.0f / (1 + sqrtf(1 + alpha2 * tan2_i));
}
```

---

## 4. 输入与输出

### `bump_alpha2`

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `N` | `float3` | 输入 | 几何法线（插值后的原始法线） |
| `Nbump` | `float3` | 输入 | 凹凸贴图扰动后的着色法线 |
| **返回值** | `float` | 输出 | $\alpha^2$ 粗糙度参数，范围 $[0, 1]$ |

### `bump_shadowing_function`

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `N` | `float3` | 输入 | 几何法线 |
| `Ld` | `float3` | 输入 | 指向光源的方向向量 |
| `alpha2` | `float` | 输入 | 由 `bump_alpha2` 计算的粗糙度参数 |
| **返回值** | `float` | 输出 | 阴影衰减因子，范围 $[0, 1]$；1 表示无遮蔽 |

---

## 5. 应用场景

### 使用方式

```cpp
// 在着色器中的典型用法：
float alpha2 = bump_alpha2(geometricNormal, shadingNormal);
float shadow = bump_shadowing_function(geometricNormal, lightDir, alpha2);
float3 finalColor = shadow * evaluateBSDF(...);
```

### 适用场景

- 任何使用法线贴图或凹凸贴图的渲染器
- 光线追踪和光栅化管线均可使用
- 对于高频法线贴图效果尤为明显

---

## 6. 关键算法深入分析

### 时间复杂度

$O(1)$ — 每个着色点仅需常数次运算。

### 数值稳定性

- `cos_d` 使用 `min(..., 1.0f)` 防止浮点误差导致超出余弦范围
- `cos_i` 使用 `max(..., 1e-6f)` 避免除零
- `clamp(0.125f * tan2_d, 0, 1)` 限制 $\alpha^2$ 范围，防止极端值

### 性能开销

两个函数总计约 10 次浮点运算 + 1 次平方根，对 GPU 性能影响极小。

---

## 7. 与其他文件的关系

本文件为独立实现。可集成到任何使用凹凸贴图的着色器中（HLSL、GLSL、CUDA 等均可移植）。

---

## 8. 在光线追踪管线中的位置

```
计算着色法线 N_bump（来自法线贴图）
       ↓
  bump_alpha2(N, N_bump)  →  α²
       ↓
  对每个光源方向 L:
    bump_shadowing_function(N, L, α²)  →  衰减因子
       ↓
  衰减因子 × BSDF 评估值 → 最终颜色贡献
```

---

## 9. 技术要点与注意事项

1. **$\frac{1}{8}$ 缩放因子**：经验选择，平衡修复效果与过度模糊。增大会使终结器区域更平滑但可能丢失细节。
2. **Smith 阴影函数**：这里使用的是 GGX/Beckmann 分布对应的高度相关 Smith 函数的简化形式。
3. **`fabsf` 的使用**：对点积取绝对值，使得函数对法线朝向不敏感，双面材质均可工作。
4. **与其他着色模型的兼容性**：返回值为标量乘数，可与任意 BRDF/BSDF 模型组合使用。

---

## 10. 扩展阅读

- **书中章节**：Chapter 12, "A Microfacet-Based Shadowing Function to Solve the Bump Terminator Problem"
- **Smith 阴影函数**：Heitz, E., "Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs," *JCGT*, 3(2), 2014.
- **GGX 分布**：Walter, B. et al., "Microfacet Models for Refraction through Rough Surfaces," *EGSR*, 2007.
