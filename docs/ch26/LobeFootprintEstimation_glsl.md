# LobeFootprintEstimation.glsl — 反射叶瓣足迹估计

> 源文件：`Ch_26_Deferred_Hybrid_Path_Tracing/LobeFootprintEstimation.glsl`

---

## 1. 文件概述

`LobeFootprintEstimation.glsl` 是一段 GLSL 代码片段，用于在延迟混合路径追踪 (Deferred Hybrid Path Tracing) 中估计反射叶瓣 (Reflection Lobe) 在屏幕空间中的足迹 (Footprint) 大小和形状。这个足迹信息用于指导空间滤波核的大小和方向，以实现高质量的降噪和重建。

---

## 2. 算法与数学背景

### 反射叶瓣足迹模型

反射叶瓣的屏幕空间足迹是一个二维椭圆区域，由两个正交方向定义：
- **反弹方向 (Bounce-off Direction)**：屏幕空间法线的投影方向
- **侧向 (Lateral Direction)**：与反弹方向垂直

### 基础尺度计算

足迹的基础尺度由粗糙度和几何关系决定：

$$\text{footprintScale} = \frac{\text{roughness} \times \text{rayLength}}{\text{rayLength} + \text{sceneZ}}$$

其中 `rayLength` 是光线传播距离，`sceneZ` 是场景深度。

### 曲率修正

凸面会缩小反射足迹（聚焦效果），凹面会扩大反射足迹（发散效果）：

$$\text{curvature} = \frac{1}{1 + \kappa_s \cdot n_z^2 \cdot \text{curvature}}$$

### 视角修正

- 沿反弹方向：$\text{footprint}_0 /= (1 - e) + e \cdot \text{NoV}$ — 掠射角时叶瓣拉伸
- 沿侧方向：$\text{footprint}_1 *= (1 - s) + s \cdot \text{NoV}$ — 掠射角时叶瓣收缩

---

## 3. 完整代码与逐行注释

```glsl
mat2 footPrint;
// "反弹"方向：屏幕空间法线 xy 分量的归一化
footPrint[0] = normalize(ssNormal.xy);
// 侧向方向：反弹方向旋转 90 度
footPrint[1] = vec2(footPrint[0].y, -footPrint[0].x);

// 基础足迹尺度：粗糙度 × 距离衰减因子
vec2 footprintScale = vec2(roughness*rayLength / (rayLength + sceneZ));

// 在凸面上，估计的足迹更小（聚焦效应）
// 构建法线坐标系的两个平面
vec3 plane0 = cross(ssV, ssNormal);         // 视线方向 × 法线
vec3 plane1 = cross(plane0, ssNormal);       // plane0 × 法线

// estimateCurvature(...) 沿 footPrint 方向从 G-Buffer 的深度
// 计算深度梯度来估计表面曲率
vec2 curvature = estimateCurvature(footPrint, plane0, plane1);

// 将曲率转换为缩放因子：曲率越大，缩放越小
curvature = 1.0 / (1.0 + CURVATURE_SCALE*square(ssNormal.z)*curvature);

// 沿两个方向分别应用曲率修正
footPrint[0] *= curvature.x;
footPrint[1] *= curvature.y;

// 确保在不同焦距相机下保持一致的尺度
footPrint *= KERNEL_FILTER / tan(cameraFov * 0.5);

// 根据 NoV（视线与法线点积）修正叶瓣拉伸/收缩
// 沿反弹方向：掠射角时叶瓣拉伸（除以较小值）
footPrint[0] /= (1.0 - ELONGATION) + ELONGATION * NoV;
// 沿侧方向：掠射角时叶瓣收缩（乘以较小值）
footPrint[1] *= (1.0 - SHRINKING) + SHRINKING * NoV;

// 使用计算好的足迹对采样点进行变换
for (i : each sample)
{
    // 将采样模式点通过足迹矩阵变换到屏幕空间
    vec2 samplingPosition = fragmentCenter + footPrint * sample[i];
    // ...
}
```

---

## 4. 输入与输出

### 输入

| 变量 | 类型 | 说明 |
|------|------|------|
| `ssNormal` | `vec3` | 屏幕空间法线 |
| `ssV` | `vec3` | 屏幕空间视线方向 |
| `roughness` | `float` | 表面粗糙度，范围 $[0, 1]$ |
| `rayLength` | `float` | 反射光线传播距离 |
| `sceneZ` | `float` | 场景深度值 |
| `NoV` | `float` | 视线与法线点积的饱和值 |
| `cameraFov` | `float` | 相机视场角（弧度） |

### 常量

| 常量 | 说明 |
|------|------|
| `CURVATURE_SCALE` | 曲率影响的缩放因子 |
| `KERNEL_FILTER` | 滤波核基础大小 |
| `ELONGATION` | 掠射角拉伸控制参数 |
| `SHRINKING` | 掠射角收缩控制参数 |

### 输出

| 变量 | 类型 | 说明 |
|------|------|------|
| `footPrint` | `mat2` | 2×2 足迹变换矩阵，将单位采样模式映射到屏幕空间椭圆 |
| `samplingPosition` | `vec2` | 各采样点在屏幕空间的位置 |

---

## 5. 应用场景

- **反射降噪**：根据足迹大小自适应调整空间滤波核，粗糙表面使用更大的核
- **自适应采样**：足迹小的区域（光滑表面、近距离）可以使用更少的采样
- **时间稳定性**：一致的足迹估计有助于减少帧间闪烁

---

## 6. 关键算法深入分析

### 距离衰减模型

$$\frac{\text{rayLength}}{\text{rayLength} + \text{sceneZ}}$$

这是一个简单而有效的近似：
- 当 `rayLength` >> `sceneZ` 时，比值趋近 1（远距离反射，大足迹）
- 当 `rayLength` << `sceneZ` 时，比值趋近 0（近距离反射，小足迹）

### mat2 作为仿射变换

`footPrint` 矩阵的两行分别是椭圆的两个半轴方向向量。通过 `footPrint * sample[i]` 将单位圆/正方形上的采样点变换到屏幕空间的椭圆区域。

### 性能考量

- 曲率估计 (`estimateCurvature`) 需要额外的 G-Buffer 纹理采样，是主要性能开销
- 其余计算均为简单的向量/矩阵运算

---

## 7. 与其他文件的关系

本文件为独立代码片段。在完整的延迟混合路径追踪管线中，足迹估计用于：
- 反射降噪 Pass 的空间滤波核计算
- 辐射度缓存查找的采样范围确定

---

## 8. 在光线追踪管线中的位置

```
G-Buffer Pass（光栅化）
  → 法线、深度、粗糙度
       ↓
反射光线追踪 Pass
  → 反射颜色、光线长度
       ↓
  ┌───────────────────────────┐
  │ LobeFootprintEstimation   │  ← 本文件
  │ 估计每个像素的采样足迹       │
  └───────────────────────────┘
       ↓
空间降噪/滤波 Pass
  → 使用足迹指导的自适应滤波
       ↓
最终合成
```

---

## 9. 技术要点与注意事项

1. **`ssNormal.z` 的作用**：在曲率修正中，`square(ssNormal.z)` 用于权衡屏幕空间曲率的可靠性——法线越接近屏幕法线（z 大），曲率估计越准确。
2. **伪代码循环**：最后的 `for (i : each sample)` 使用伪代码语法，实际实现中需要替换为具体的循环构造。
3. **`estimateCurvature` 未定义**：此函数在代码片段中未给出实现，它通过有限差分方法从 G-Buffer 深度计算表面曲率。
4. **相机 FoV 归一化**：除以 `tan(cameraFov * 0.5)` 确保不同焦距下足迹大小一致。

---

## 10. 扩展阅读

- **书中章节**：Chapter 26, "Deferred Hybrid Path Tracing"
- **SVGF 降噪**：Schied, C. et al., "Spatiotemporal Variance-Guided Filtering: Real-Time Reconstruction for Path-Traced Global Illumination," *HPG*, 2017.
- **叶瓣追踪**：Xu, K. et al., "Anisotropic Spherical Gaussians," *SIGGRAPH Asia*, 2013.
