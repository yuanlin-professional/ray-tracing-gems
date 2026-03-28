# edgeoutline.cu 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/edgeoutline.cu`

本文件实现了**边缘轮廓描边 (Edge Outline)** 着色效果，通过在几何体边缘（法线与视线方向接近垂直的区域）添加暗色轮廓线来增强物体的视觉可辨识度。这种技术在科学可视化中特别重要——当大量球体（如原子模型）紧密排列时，没有轮廓线的球体边界难以区分。

## 算法与数学背景

### 边缘因子计算

边缘检测基于**视线方向 (View Direction)** 与**表面法线 (Surface Normal)** 的点积：

$$e = (\mathbf{N} \cdot \mathbf{D})^2$$

其中 $\mathbf{N}$ 为法线，$\mathbf{D}$ 为射线方向。

将其反转并进行幂次映射：

$$e' = 1 - (1 - e)^{(1 - w) \cdot 32}$$

其中 $w$ 为轮廓宽度参数 (`outlinewidth`)，控制轮廓线的粗细。

最终混合因子：

$$f = \text{saturate}((1 - o) + e' \cdot o)$$

其中 $o$ 为轮廓强度参数 (`outline`)。

### 边缘检测原理

在表面边缘处，$\mathbf{N} \cdot \mathbf{D} \approx 0$（法线与视线接近垂直），因此：
- $e \approx 0$
- $1 - e \approx 1$
- $(1-e)^{k} \approx 1$（对于大 $k$）
- $e' = 1 - 1 = 0$
- $f \approx (1 - o)$，颜色被显著压暗

在表面中心处，$\mathbf{N} \cdot \mathbf{D} \approx \pm1$，因此：
- $e \approx 1$
- $e' \approx 1$
- $f \approx 1$，颜色不受影响

## 代码结构概览

由于本文件较短（<30 行核心逻辑），以下提供完整源码及逐行注释。

## 逐段代码详解

```cuda
// 第1-3行：功能说明
// This example code snippet adds a dark outline on the edges
// of geometry to help accentuate objects that are packed
// closely together and may not otherwise be visually distinct.
```

### 射线负载结构

```cuda
struct PerRayData_radiance {
  float3 result;     // Final shaded surface color
  // ...
}

rtDeclareVariable(PerRayData_radiance, prd, rtPayload, );
```

### 最近命中着色器（含描边）

```cuda
// 第13行：启用描边的着色器变体
RT_PROGRAM void closest_hit_shader_outline( ... ) {
  // Skipping boilerplate closest-hit shader material here ...
```

注意函数名为 `closest_hit_shader_outline`，这是一个专门的着色器变体，表明 Tachyon 为描边材质使用了独立的着色程序。

```cuda
  // 第17行：检查是否启用描边
  if (outline > 0.0f) {
```
`outline` 参数范围 $[0, 1]$，0 表示无描边，1 表示最大描边强度。

```cuda
    // 第18行：计算边缘因子（N·D 的平方）
    float edgefactor = dot(N, ray.direction);
    edgefactor *= edgefactor;
```
$e = (\mathbf{N} \cdot \mathbf{D})^2$。取平方使函数关于法线对称（无论从正面还是背面看，效果相同）。

```cuda
    // 第20行：反转——边缘处值为 1，中心为 0
    edgefactor = 1.0f - edgefactor;
```
$1 - e$：现在边缘处值接近 1，中心接近 0。

```cuda
    // 第21行：幂次映射，控制轮廓宽度
    edgefactor = 1.0f - powf(edgefactor, (1.0f - outlinewidth) * 32.0f);
```
$e' = 1 - (1 - e)^{(1 - w) \cdot 32}$

- `outlinewidth = 0`：指数 = 32，轮廓非常窄（只有极端边缘变暗）
- `outlinewidth = 0.5`：指数 = 16，中等宽度
- `outlinewidth = 1.0`：指数 = 0，所有区域均匀（无效果，因为 $x^0 = 1$）

```cuda
    // 第22行：混合轮廓强度
    float outlinefactor = __saturatef((1.0f - outline) + (edgefactor * outline));
```
$f = \text{clamp}((1 - o) + e' \cdot o, 0, 1)$

- `outline = 0`：$f = 1$，无描边
- `outline = 1`：$f = e'$，全强度描边

```cuda
    // 第23行：应用到最终颜色
    result *= outlinefactor;
  }
```
将颜色乘以轮廓因子——边缘处变暗。

```cuda
  prd.result = result; // Pass the resulting color back up the tree.
}
```

## 关键算法深入分析

### 参数空间可视化

假设表面上的点，其法线与视线的夹角为 $\theta$：

```
θ = 0° (正对)    θ = 45°         θ = 90° (边缘)
N·D = 1          N·D ≈ 0.707     N·D = 0
e = 1            e ≈ 0.5          e = 0
1-e = 0          1-e ≈ 0.5        1-e = 1

outlinewidth=0 (窄):
(1-e)^32:  ≈0    ≈2.3e-10        1.0
e' = 1-x:  ≈1    ≈1.0             0.0
效果:      亮     亮               暗  ← 仅极端边缘变暗

outlinewidth=0.5 (中):
(1-e)^16:  ≈0    ≈1.5e-5         1.0
e' = 1-x:  ≈1    ≈1.0             0.0
效果:      亮     亮               暗  ← 边缘较宽变暗区

outlinewidth=0.9 (宽):
(1-e)^3.2: ≈0    ≈0.1             1.0
e' = 1-x:  ≈1    ≈0.9             0.0
效果:      亮     略暗             暗  ← 大范围变暗
```

### 与传统描边技术的比较

| 技术 | 方法 | 优点 | 缺点 |
|------|------|------|------|
| 本实现 | N·D 边缘检测 | 简单、逐像素、无需额外 pass | 仅检测轮廓倾斜，无法检测折痕 |
| 后处理边缘 | 深度/法线差异 | 检测所有类型边缘 | 需要额外 pass |
| 几何外扩 | 两次渲染 | 一致的线宽 | 性能开销大 |

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `N` | `float3` | 表面法线 |
| `ray.direction` | `float3` | 射线方向 |
| `outline` | `float` | 描边强度 $[0, 1]$ |
| `outlinewidth` | `float` | 描边宽度 $[0, 1]$ |
| `result` | `float3` | 着色前的颜色 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `prd.result` | `float3` | 应用描边后的最终颜色 |

## 与其他文件的关系

- **`aomaxdist.cu`**：AO 与描边可以叠加使用，增强空间感
- **`transedge.cu`**：角度依赖透明度使用类似的 N·D 计算
- **`onv-packing.cu`**：提供压缩法线的解码功能

## 在光线追踪管线中的位置

```
主射线 → BVH 遍历 → 最近命中着色器 → 基础着色 → [边缘描边] → 输出颜色
                                                  ^^^^^^^^^^^^
                                                  本文件实现此步骤
```

描边效果在最近命中着色器的末尾（基础着色之后）应用，作为颜色的乘性修正。

## 技术要点与注意事项

1. **`__saturatef()`**：CUDA 的饱和函数，将结果钳制到 $[0, 1]$，是 GPU 上的原生操作
2. **平方的必要性**：`edgefactor *= edgefactor` 确保正面和背面的处理对称（$\cos\theta$ 可能为负值）
3. **着色器变体**：使用独立的着色器函数 `closest_hit_shader_outline` 而非运行时分支，避免 GPU 上的线程发散
4. **幂次计算开销**：`powf()` 在 GPU 上相对昂贵，但每像素只计算一次，不是性能瓶颈
5. **与阴影的交互**：描边在阴影计算之后应用可能导致阴影区域的边缘不够明显

## 扩展阅读

- Gooch, B. et al. *A Non-Photorealistic Lighting Model for Automatic Technical Illustration*. SIGGRAPH 1998
- Saito, T., Takahashi, T. *Comprehensible Rendering of 3-D Shapes*. SIGGRAPH 1990
- Stone, J.E. et al. *Immersive Molecular Visualization with Omnidirectional Stereoscopic Ray Tracing and Remote Rendering*. HPDC 2016
