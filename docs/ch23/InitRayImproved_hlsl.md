# InitRayImproved.hlsl 技术文档

## 文件概述

`InitRayImproved.hlsl` 是 Frostbite 引擎中交互式光照贴图预览系统的**改进版路径追踪光照积分内核**。相较于基础版 `InitRay.hlsl`，本文件引入了两项关键改进：
1. **余弦加权半球采样 (Cosine-Weighted Hemisphere Sampling)**：替代均匀半球采样，降低方差
2. **下一事件估计 / 局部光源直接采样 (Next Event Estimation / Direct Light Sampling)**：在每个路径顶点显式采样局部光源

这两项改进显著加速了渐进式光照贴图的收敛速度。

**源文件路径**: `Ch_23_Interactive Light Map and Irradiance Volume Preview in Frostbite/InitRayImproved.hlsl`

## 算法与数学背景

### 余弦加权半球采样

基础版使用均匀半球采样，其 PDF 为 $p(\omega) = \frac{1}{2\pi}$。改进版使用余弦加权半球采样，PDF 为：

$$p(\omega) = \frac{\cos\theta}{\pi}$$

这与 Lambert BRDF 的形式完美匹配。对于 Lambert 漫反射表面：

$$\frac{f_r \cdot \cos\theta}{p(\omega)} = \frac{\frac{\rho}{\pi} \cdot \cos\theta}{\frac{\cos\theta}{\pi}} = \rho$$

因此在余弦加权采样下，每次弹射的权重简化为仅剩反照率 $\rho$，消除了显式的余弦项，从而大幅降低估计器方差。

### 下一事件估计 (Next Event Estimation, NEE)

基础版仅依靠随机弹射来"碰中"光源。改进版在每个路径顶点处，调用 `sampleLocalLighting()` 直接对局部光源 (点光源、聚光灯等) 进行采样。这使得直接光照贡献可以在每次弹射中被高效计算，而不是仅靠随机碰撞。

两条采样策略（间接弹射 + 直接光源采样）的组合即为所谓的**多重重要性采样 (Multiple Importance Sampling, MIS)** 框架的简化实现。

## 代码结构概览

```
初始化光线 → 循环追踪 → [未命中→天空] → 计算命中点位置
  → 直接采样局部光源 → 更新吞吐量 → 余弦加权半球弹射 → 返回结果
```

## 完整源码与逐行注释

```hlsl
// Kernel code describing the lighting integration. Improved version.
// 内核代码：光照积分的改进版本

// 从纹素原点沿随机方向初始化光线（与基础版相同）
Ray r = initRay(texelOrigin, randomDirection);

// 输出辐射度，初始化为 0
float3 outRadiance = 0;
// 路径吞吐量，初始化为 1
float3 pathThroughput = 1;

// 主循环：路径追踪
while (pathVertexCount++ < maxDepth) {
    // 光线追踪结果
    PrimaryRayData rayData;
    TraceRay(r, rayData);

    // 光线未命中任何几何体 → 采样天空穹顶并终止路径
    if (!rayData.hasHitAnything) {
        outRadiance += pathThroughput * getSkyDome(r.Direction);
        break;
    }

    // [改进1] 计算命中点的世界空间位置
    float3 Pos = r.Origin + r.Direction * rayData.hitT;
    // [改进2] 在命中点直接采样局部光源（下一事件估计 NEE）
    // sampleLocalLighting 返回从该点到所有局部光源的直接辐射度贡献
    float3 L = sampleLocalLighting(Pos, rayData.Normal);

    // [改进3] 路径吞吐量仅乘以 albedo（无需显式 cos 项）
    // 因为余弦加权半球采样的 PDF 已经包含了 cos(theta)/pi
    // 与 Lambert BRDF 的 rho/pi 抵消后只剩 rho (即 albedo)
    pathThroughput *= rayData.albedo;
    // 累积辐射度：直接光照 L + 自发光 emissive，都乘以吞吐量
    outRadiance += pathThroughput * (L + rayData.emissive);

    // 更新光线原点
    r.Origin = Pos;
    // [改进4] 使用余弦加权半球采样（替代均匀半球采样）
    r.Direction = sampleCosineHemisphere(rayData.Normal);
}

// 返回累积的总辐射度
return outRadiance;
```

## 关键算法深入分析

### 基础版与改进版的对比

| 特性 | InitRay.hlsl (基础版) | InitRayImproved.hlsl (改进版) |
|------|----------------------|-------------------------------|
| 半球采样 | `sampleHemisphere()` 均匀采样 | `sampleCosineHemisphere()` 余弦加权 |
| 直接光照 | 仅通过随机弹射命中 | `sampleLocalLighting()` 显式采样 |
| 吞吐量更新 | `albedo * cos(theta)` | 仅 `albedo` |
| 收敛速度 | 较慢 | 显著更快 |
| 方差 | 较高 | 较低 |

### 为什么吞吐量更新中不再需要 cos(theta)

在基础版中：
$$\text{weight} = \frac{f_r \cdot \cos\theta}{p_{\text{uniform}}} = \frac{\frac{\rho}{\pi} \cdot \cos\theta}{\frac{1}{2\pi}} = 2\rho \cos\theta$$

在改进版中：
$$\text{weight} = \frac{f_r \cdot \cos\theta}{p_{\text{cosine}}} = \frac{\frac{\rho}{\pi} \cdot \cos\theta}{\frac{\cos\theta}{\pi}} = \rho$$

### 直接光源采样的贡献

`sampleLocalLighting(Pos, Normal)` 函数计算从命中点 `Pos` 对所有局部光源的直接照明贡献。这包括：
- 对每个光源采样一个阴影射线 (shadow ray) 判断可见性
- 计算光源的辐射度贡献并按距离衰减

将直接光照 `L` 与自发光 `rayData.emissive` 一同加入 `outRadiance`，乘以当前路径吞吐量。

## 输入与输出

| 参数 | 类型 | 说明 |
|------|------|------|
| `texelOrigin` | `float3` | 光照贴图纹素在世界空间中的位置 |
| `randomDirection` | `float3` | 初始随机方向 |
| `maxDepth` | `int` | 最大路径弹射深度 |
| **返回值** | `float3` | 累积的出射辐射度 (RGB) |

## 与其他文件的关系

- **InitRay.hlsl**: 本文件的基础版本，用于对比展示优化效果
- **PerformanceBudgeting.hlsl**: 控制每帧执行本内核的纹素数量
- **VarianceTracking.hlsl**: 跟踪本内核输出结果的方差，判断收敛

## 在光线追踪管线中的位置

与基础版相同，运行在 DXR 管线的 Ray Generation 阶段。改进版在每个路径顶点额外发射阴影射线 (shadow ray) 用于直接光源采样，因此实际的光线追踪调用次数更多，但每个样本的信息量更大，总体收敛更快。

```
[Ray Generation]
  ├── TraceRay (间接弹射) ──> [Closest Hit] ──> 返回命中信息
  └── TraceRay (阴影射线, sampleLocalLighting 内部) ──> [Any Hit] ──> 可见性
```

## 技术要点与注意事项

1. **余弦加权采样的实现**：`sampleCosineHemisphere()` 通常通过 Malley 方法实现 —— 先在单位圆盘上均匀采样，再投影到半球
2. **能量守恒**：改进版通过正确的 PDF 与 BRDF 配对确保无偏 (unbiased) 估计
3. **光源遮挡**：`sampleLocalLighting()` 内部需要发射阴影射线来判断光源是否被遮挡
4. **自发光与直接光照分离**：注意 `emissive` 和 `L` 是在同一层级相加的，都受 `pathThroughput` 调制
5. **无环境光特殊处理**：天空穹顶仍然仅在光线 miss 时采样，未对环境光做 NEE

## 扩展阅读

- Shirley, P. *Ray Tracing in One Weekend* 系列 —— 余弦加权采样的推导
- Veach, E. & Guibas, L. *Optimally Combining Sampling Techniques for Monte Carlo Rendering*, SIGGRAPH 1995
- Ray Tracing Gems, Chapter 23, Section 23.5: Improved Light Integration
