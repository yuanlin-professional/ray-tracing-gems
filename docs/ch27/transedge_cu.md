# transedge.cu 技术文档

## 文件概述

**文件路径**: `Ch_27_Interactive_Ray_Tracing_Techniques_for_High-Fidelity_Scientific_Visualization/transedge.cu`

本文件实现了**角度依赖的透明度 (Angle-Dependent Transparency)** 效果，模拟 Tachyon/Raster3D 风格的"X 光视图"渲染。该效果使正对观察者的表面变得几乎完全透明，而侧视的表面（边缘区域）则保持较高的不透明度。这种技术在科学可视化中被广泛使用——例如查看蛋白质复合体内部结构时，外层分子表面变透明，但其轮廓仍然可见。

## 算法与数学背景

### 角度依赖透明度公式

给定基础透明度参数 $\alpha_0$ 和视线方向与法线的夹角，最终透明度为：

$$\alpha = \left[1 + \cos\left(\pi (1 - \alpha_0) \cdot (\mathbf{N} \cdot \mathbf{D})\right)\right]^2 \cdot 0.25$$

其中：
- $\mathbf{N}$ 为表面法线
- $\mathbf{D}$ 为射线方向
- $\alpha_0$ 为材质的基础不透明度参数

### 函数特性分析

当 $\mathbf{N} \cdot \mathbf{D} = 0$（边缘，法线垂直于视线）：

$$\alpha = [1 + \cos(0)]^2 \cdot 0.25 = [1 + 1]^2 \cdot 0.25 = 1.0$$

表面完全不透明（边缘清晰可见）。

当 $\mathbf{N} \cdot \mathbf{D} = \pm1$（正对，法线平行于视线），且 $\alpha_0$ 接近 0：

$$\alpha = [1 + \cos(\pi)]^2 \cdot 0.25 = [1 - 1]^2 \cdot 0.25 = 0.0$$

表面完全透明（可以看穿）。

## 代码结构概览

由于本文件较短（<30 行核心逻辑），以下提供完整源码及逐行注释。

## 逐段代码详解

```cuda
// 第1-5行：功能说明
// This example code snippet makes viewer-facing surfaces appear
// completely transparent while leaving surfaces seen edge-on
// more visible and opaque. This type of rendering is extremely
// useful to facilitate views into the interior of crowded
// scenes, such as densely packed biomolecular complexes.
```

### 最近命中着色器

```cuda
RT_PROGRAM void closest_hit_shader( ... ) {
  // Skipping boilerplate closest-hit shader material here ...
```

```cuda
  // 第11行：检查是否为透明表面
  if (alpha < 0.999f) {
```
仅对不透明度 (opacity) 小于 0.999 的材质应用透明效果。

```cuda
    // 第13-16行：角度依赖透明度计算
    if (transmode) {
      alpha = 1.0f + cosf(3.1415926f * (1.0f-alpha) * dot(N, ray.direction));
      alpha = alpha*alpha * 0.25f;
    }
```

逐行分析：
1. `transmode`：全局开关，启用角度依赖透明度模式
2. `3.1415926f * (1.0f-alpha)`：$\pi \cdot (1 - \alpha_0)$，将基础不透明度映射到 $[0, \pi]$ 范围
3. `dot(N, ray.direction)`：$\mathbf{N} \cdot \mathbf{D}$，法线与视线的余弦值
4. `cosf(...)`：余弦函数提供平滑的角度过渡
5. `1.0f + cosf(...)`：将余弦值从 $[-1, 1]$ 映射到 $[0, 2]$
6. `alpha*alpha * 0.25f`：平方并缩放，将 $[0, 2]$ 映射到 $[0, 1]$

```cuda
    // 第17行：按新透明度缩放当前光照
    result *= alpha;
```
将当前表面的着色结果乘以透明度——透明区域的贡献减小。

```cuda
    // 第19-20行：发射透射射线
    // Skipping boilerplate code to prepare a new transmission ray ...
    rtTrace(root_object, trans_ray, new_prd);
  }
```
追踪透射射线 (Transmission Ray)，获取表面背后的颜色。

```cuda
  // 第22行：混合当前表面和透射颜色
  result += (1.0f - alpha) * new_prd.result;
```
经典的 Alpha 混合 (Alpha Blending)：

$$\text{result} = \alpha \cdot \text{surface\_color} + (1 - \alpha) \cdot \text{transmission\_color}$$

```cuda
  prd.result = result; // Pass the resulting color back up the tree.
}
```

## 关键算法深入分析

### 透明度响应曲线

对于不同的 $\alpha_0$ 值，透明度随观察角度 $\theta$ 的变化：

```
不透明度 α
1.0 ┤ ●─────────────────────────●
    │  \                       /
    │   \   α₀ = 0.3         /
    │    \                   /
0.5 ┤     \                 /
    │      \               /
    │       \   α₀ = 0.1  /
    │        \           /
0.0 ┤─────────●─────────●───────
    90°       45°        0°       观察角 θ
   (边缘)               (正对)
```

- $\alpha_0 = 0$：完全透明模式，正对表面完全透明
- $\alpha_0 = 0.5$：半透明模式，正对表面略有可见
- $\alpha_0 = 1.0$：完全不透明，`transmode` 无效

### 科学可视化应用场景

1. **分子复合体**：外层蛋白质表面透明，内部配体清晰可见
2. **细胞膜**：膜表面透明，内部细胞器显露
3. **密集粒子系统**：前排粒子透明，后排粒子可见

### 与 edgeoutline.cu 的比较

| 特性 | `edgeoutline.cu` | `transedge.cu` |
|------|-------------------|----------------|
| 原理 | 边缘变暗 | 正面变透明 |
| 效果 | 不透明轮廓线 | 透明带边缘保留 |
| 适用对象 | 不透明物体 | 透明物体 |
| 递归射线 | 不需要 | 需要透射射线 |

两者可以组合使用以获得更丰富的视觉效果。

## 输入与输出

### 输入
| 名称 | 类型 | 说明 |
|------|------|------|
| `alpha` | `float` | 材质基础不透明度 $[0, 1]$ |
| `transmode` | `bool` | 是否启用角度依赖透明度 |
| `N` | `float3` | 表面法线 |
| `ray.direction` | `float3` | 入射射线方向 |
| `result` | `float3` | 当前表面着色颜色 |

### 输出
| 名称 | 类型 | 说明 |
|------|------|------|
| `prd.result` | `float3` | 混合后的最终颜色 |

## 与其他文件的关系

- **`transmax.cu`**：限制透射射线递归深度，防止无限递归
- **`edgeoutline.cu`**：类似的 N-dot-D 角度检测，但产生描边效果而非透明效果
- **`clipping.cpp`**：裁剪平面与透明度结合可提供截面视图
- **`aomaxdist.cu`**：透明物体的 AO 计算需要特殊处理

## 在光线追踪管线中的位置

```
主射线 → 最近命中着色器 → [角度依赖透明度计算]
                              │
                              ├── alpha ≈ 0: 几乎全透明 → 发射透射射线
                              │                            │
                              │                            v
                              │                      递归追踪
                              │
                              └── alpha ≈ 1: 不透明 → 直接输出颜色

最终: result = α * surface + (1-α) * transmission
```

## 技术要点与注意事项

1. **递归深度**：透射射线导致递归，需要配合 `transmax.cu` 中的最大穿越限制
2. **性能开销**：每个透明表面都会发射额外射线，密集透明场景性能开销大
3. **余弦函数精度**：使用 `3.1415926f` 作为 $\pi$ 的近似值足以满足渲染精度需求
4. **N-dot-D 符号**：`dot(N, ray.direction)` 对于正面命中通常为负值，对于背面为正值。余弦函数的偶数次幂使结果对称
5. **颜色累积**：透射链上的颜色采用"over"合成操作，需注意能量守恒

## 扩展阅读

- Merritt, E.A., Bacon, D.J. *Raster3D: Photorealistic Molecular Graphics*. Methods in Enzymology, 1997
- Stone, J.E. *An Efficient Library for Parallel Ray Tracing and Animation*. Master's Thesis, 1998
- Porter, T., Duff, T. *Compositing Digital Images*. SIGGRAPH 1984
